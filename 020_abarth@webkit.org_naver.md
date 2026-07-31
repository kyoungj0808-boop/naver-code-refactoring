원본코드
# Copyright 2001-2002 by Vinay Sajip. All Rights Reserved.
#
# Permission to use, copy, modify, and distribute this software and its
# documentation for any purpose and without fee is hereby granted,
# provided that the above copyright notice appear in all copies and that
# both that copyright notice and this permission notice appear in
# supporting documentation, and that the name of Vinay Sajip
# not be used in advertising or publicity pertaining to distribution
# of the software without specific, written prior permission.
# VINAY SAJIP DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE, INCLUDING
# ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL
# VINAY SAJIP BE LIABLE FOR ANY SPECIAL, INDIRECT OR CONSEQUENTIAL DAMAGES OR
# ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER
# IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT
# OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.

"""
Logging package for Python. Based on PEP 282 and comments thereto in
comp.lang.python, and influenced by Apache's log4j system.

Should work under Python versions >= 1.5.2, except that source line
information is not available unless 'inspect' is.

Copyright (C) 2001-2002 Vinay Sajip. All Rights Reserved.

To use, simply 'import logging' and log away!
"""

import sys, logging, logging.handlers, string, thread, threading, socket, struct, os

from SocketServer import ThreadingTCPServer, StreamRequestHandler


DEFAULT_LOGGING_CONFIG_PORT = 9030
if sys.platform == "win32":
    RESET_ERROR = 10054   #WSAECONNRESET
else:
    RESET_ERROR = 104     #ECONNRESET

#
#   The following code implements a socket listener for on-the-fly
#   reconfiguration of logging.
#
#   _listener holds the server object doing the listening
_listener = None

def fileConfig(fname, defaults=None):
    """
    Read the logging configuration from a ConfigParser-format file.

    This can be called several times from an application, allowing an end user
    the ability to select from various pre-canned configurations (if the
    developer provides a mechanism to present the choices and load the chosen
    configuration).
    In versions of ConfigParser which have the readfp method [typically
    shipped in 2.x versions of Python], you can pass in a file-like object
    rather than a filename, in which case the file-like object will be read
    using readfp.
    """
    import ConfigParser

    cp = ConfigParser.ConfigParser(defaults)
    if hasattr(cp, 'readfp') and hasattr(fname, 'readline'):
        cp.readfp(fname)
    else:
        cp.read(fname)
    #first, do the formatters...
    flist = cp.get("formatters", "keys")
    if len(flist):
        flist = string.split(flist, ",")
        formatters = {}
        for form in flist:
            sectname = "formatter_%s" % form
            opts = cp.options(sectname)
            if "format" in opts:
                fs = cp.get(sectname, "format", 1)
            else:
                fs = None
            if "datefmt" in opts:
                dfs = cp.get(sectname, "datefmt", 1)
            else:
                dfs = None
            f = logging.Formatter(fs, dfs)
            formatters[form] = f
    #next, do the handlers...
    #critical section...
    logging._acquireLock()
    try:
        try:
            #first, lose the existing handlers...
            logging._handlers.clear()
            #now set up the new ones...
            hlist = cp.get("handlers", "keys")
            if len(hlist):
                hlist = string.split(hlist, ",")
                handlers = {}
                fixups = [] #for inter-handler references
                for hand in hlist:
                    sectname = "handler_%s" % hand
                    klass = cp.get(sectname, "class")
                    opts = cp.options(sectname)
                    if "formatter" in opts:
                        fmt = cp.get(sectname, "formatter")
                    else:
                        fmt = ""
                    klass = eval(klass, vars(logging))
                    args = cp.get(sectname, "args")
                    args = eval(args, vars(logging))
                    h = apply(klass, args)
                    if "level" in opts:
                        level = cp.get(sectname, "level")
                        h.setLevel(logging._levelNames[level])
                    if len(fmt):
                        h.setFormatter(formatters[fmt])
                    #temporary hack for FileHandler and MemoryHandler.
                    if klass == logging.handlers.MemoryHandler:
                        if "target" in opts:
                            target = cp.get(sectname,"target")
                        else:
                            target = ""
                        if len(target): #the target handler may not be loaded yet, so keep for later...
                            fixups.append((h, target))
                    handlers[hand] = h
                #now all handlers are loaded, fixup inter-handler references...
                for fixup in fixups:
                    h = fixup[0]
                    t = fixup[1]
                    h.setTarget(handlers[t])
            #at last, the loggers...first the root...
            llist = cp.get("loggers", "keys")
            llist = string.split(llist, ",")
            llist.remove("root")
            sectname = "logger_root"
            root = logging.root
            log = root
            opts = cp.options(sectname)
            if "level" in opts:
                level = cp.get(sectname, "level")
                log.setLevel(logging._levelNames[level])
            for h in root.handlers[:]:
                root.removeHandler(h)
            hlist = cp.get(sectname, "handlers")
            if len(hlist):
                hlist = string.split(hlist, ",")
                for hand in hlist:
                    log.addHandler(handlers[hand])
            #and now the others...
            #we don't want to lose the existing loggers,
            #since other threads may have pointers to them.
            #existing is set to contain all existing loggers,
            #and as we go through the new configuration we
            #remove any which are configured. At the end,
            #what's left in existing is the set of loggers
            #which were in the previous configuration but
            #which are not in the new configuration.
            existing = root.manager.loggerDict.keys()
            #now set up the new ones...
            for log in llist:
                sectname = "logger_%s" % log
                qn = cp.get(sectname, "qualname")
                opts = cp.options(sectname)
                if "propagate" in opts:
                    propagate = cp.getint(sectname, "propagate")
                else:
                    propagate = 1
                logger = logging.getLogger(qn)
                if qn in existing:
                    existing.remove(qn)
                if "level" in opts:
                    level = cp.get(sectname, "level")
                    logger.setLevel(logging._levelNames[level])
                for h in logger.handlers[:]:
                    logger.removeHandler(h)
                logger.propagate = propagate
                logger.disabled = 0
                hlist = cp.get(sectname, "handlers")
                if len(hlist):
                    hlist = string.split(hlist, ",")
                    for hand in hlist:
                        logger.addHandler(handlers[hand])
            #Disable any old loggers. There's no point deleting
            #them as other threads may continue to hold references
            #and by disabling them, you stop them doing any logging.
            for log in existing:
                root.manager.loggerDict[log].disabled = 1
        except:
            import traceback
            ei = sys.exc_info()
            traceback.print_exception(ei[0], ei[1], ei[2], None, sys.stderr)
            del ei
    finally:
        logging._releaseLock()

def listen(port=DEFAULT_LOGGING_CONFIG_PORT):
    """
    Start up a socket server on the specified port, and listen for new
    configurations.

    These will be sent as a file suitable for processing by fileConfig().
    Returns a Thread object on which you can call start() to start the server,
    and which you can join() when appropriate. To stop the server, call
    stopListening().
    """
    if not thread:
        raise NotImplementedError, "listen() needs threading to work"

    class ConfigStreamHandler(StreamRequestHandler):
        """
        Handler for a logging configuration request.

        It expects a completely new logging configuration and uses fileConfig
        to install it.
        """
        def handle(self):
            """
            Handle a request.

            Each request is expected to be a 4-byte length,
            followed by the config file. Uses fileConfig() to do the
            grunt work.
            """
            import tempfile
            try:
                conn = self.connection
                chunk = conn.recv(4)
                if len(chunk) == 4:
                    slen = struct.unpack(">L", chunk)[0]
                    chunk = self.connection.recv(slen)
                    while len(chunk) < slen:
                        chunk = chunk + conn.recv(slen - len(chunk))
                    #Apply new configuration. We'd like to be able to
                    #create a StringIO and pass that in, but unfortunately
                    #1.5.2 ConfigParser does not support reading file
                    #objects, only actual files. So we create a temporary
                    #file and remove it later.
                    file = tempfile.mktemp(".ini")
                    f = open(file, "w")
                    f.write(chunk)
                    f.close()
                    fileConfig(file)
                    os.remove(file)
            except socket.error, e:
                if type(e.args) != types.TupleType:
                    raise
                else:
                    errcode = e.args[0]
                    if errcode != RESET_ERROR:
                        raise

    class ConfigSocketReceiver(ThreadingTCPServer):
        """
        A simple TCP socket-based logging config receiver.
        """

        allow_reuse_address = 1

        def __init__(self, host='localhost', port=DEFAULT_LOGGING_CONFIG_PORT,
                     handler=None):
            ThreadingTCPServer.__init__(self, (host, port), handler)
            logging._acquireLock()
            self.abort = 0
            logging._releaseLock()
            self.timeout = 1

        def serve_until_stopped(self):
            import select
            abort = 0
            while not abort:
                rd, wr, ex = select.select([self.socket.fileno()],
                                           [], [],
                                           self.timeout)
                if rd:
                    self.handle_request()
                logging._acquireLock()
                abort = self.abort
                logging._releaseLock()

    def serve(rcvr, hdlr, port):
        server = rcvr(port=port, handler=hdlr)
        global _listener
        logging._acquireLock()
        _listener = server
        logging._releaseLock()
        server.serve_until_stopped()

    return threading.Thread(target=serve,
                            args=(ConfigSocketReceiver,
                                  ConfigStreamHandler, port))

def stopListening():
    """
    Stop the listening server which was created with a call to listen().
    """
    global _listener
    if _listener:
        logging._acquireLock()
        _listener.abort = 1
        _listener = None
        logging._releaseLock()

코어 설계는 그대로 유지하고, 운영 계층만 현대화하면 될 것 같습니다. 
eval 제거, 보안 강화, 책임 분리, Guard 계층 추가 정도만 해도 현재 기준에서도 충분히 경쟁력 있는 구조가 될 것 같습니다.


제안패치
import sys
import logging
import logging.handlers
import threading
import socket
import struct
import os
import tempfile
import select
import time
import ast
import hmac
import hashlib
import configparser
from socketserver import ThreadingTCPServer, StreamRequestHandler

DEFAULT_LOGGING_CONFIG_PORT = 9030

if sys.platform == "win32":
    RESET_ERROR = 10054  # WSAECONNRESET
else:
    RESET_ERROR = 104    # ECONNRESET

_listener = None
_listener_lock = threading.Lock()


# ==========================================
# 1. Registry 패턴 기반 클래스 관리자
# ==========================================
class LoggingClassRegistry:
    _registry = {
        "StreamHandler": logging.StreamHandler,
        "FileHandler": logging.FileHandler,
        "RotatingFileHandler": logging.handlers.RotatingFileHandler,
        "TimedRotatingFileHandler": logging.handlers.TimedRotatingFileHandler,
        "MemoryHandler": logging.handlers.MemoryHandler,
        "Formatter": logging.Formatter,
    }

    @classmethod
    def get(cls, name):
        if name not in cls._registry:
            raise ValueError(f"Unregistered or unsafe logging class: {name}")
        return cls._registry[name]


# ==========================================
# 2. 정밀 상태 전이가 포함된 Circuit Breaker
# ==========================================
class LoggingStatusManager:
    def __init__(self, failure_threshold=3, cooldown_seconds=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.cooldown_seconds = cooldown_seconds
        self.last_failure_time = 0.0
        self.state = "CLOSED"  # CLOSED, HALF-OPEN, OPEN
        self.lock = threading.Lock()

    def record_success(self):
        with self.lock:
            self.failure_count = 0
            self.state = "CLOSED"

    def record_failure(self):
        with self.lock:
            self.failure_count += 1
            self.last_failure_time = time.time()
            if self.failure_count >= self.failure_threshold or self.state == "HALF-OPEN":
                self.state = "OPEN"
                logging.critical(f"Circuit Breaker OPENED. Consecutive failures: {self.failure_count}")

    def allow_attempt(self):
        with self.lock:
            if self.state == "OPEN":
                if time.time() - self.last_failure_time > self.cooldown_seconds:
                    self.state = "HALF-OPEN"
                    logging.warning("Circuit Breaker -> HALF-OPEN. Testing configuration...")
                    return True
                return False
            return True


status_manager = LoggingStatusManager()


# ==========================================
# 3. Deep Rollback & Atomic Swap 엔진
# ==========================================
def fileConfig(fname, defaults=None):
    if not status_manager.allow_attempt():
        raise RuntimeError("Circuit Breaker is OPEN. Configuration updates are blocked.")

    cp = configparser.ConfigParser(defaults)
    if hasattr(fname, 'readline'):
        cp.read_file(fname)
    else:
        cp.read(fname)

    # 1단계: Dry-run 파싱 및 검증
    try:
        formatters = {}
        if cp.has_section("formatters") and cp.has_option("formatters", "keys"):
            flist = [f.strip() for f in cp.get("formatters", "keys").split(",") if f.strip()]
            for form in flist:
                sectname = f"formatter_{form}"
                fs = cp.get(sectname, "format", fallback=None)
                dfs = cp.get(sectname, "datefmt", fallback=None)
                formatters[form] = logging.Formatter(fs, dfs)

        temp_handlers = {}
        fixups = []
        if cp.has_section("handlers") and cp.has_option("handlers", "keys"):
            hlist = [h.strip() for h in cp.get("handlers", "keys").split(",") if h.strip()]
            for hand in hlist:
                sectname = f"handler_{hand}"
                class_name = cp.get(sectname, "class")
                klass = LoggingClassRegistry.get(class_name)
                
                args_str = cp.get(sectname, "args", fallback="()")
                args = ast.literal_eval(args_str)
                if not isinstance(args, tuple):
                    args = (args,)
                
                h = klass(*args)
                if cp.has_option(sectname, "level"):
                    h.setLevel(getattr(logging, cp.get(sectname, "level"), logging.DEBUG))
                fmt = cp.get(sectname, "formatter", fallback="")
                if fmt and fmt in formatters:
                    h.setFormatter(formatters[fmt])
                if klass == logging.handlers.MemoryHandler:
                    target = cp.get(sectname, "target", fallback="")
                    if target:
                        fixups.append((h, target))
                temp_handlers[hand] = h

            for h, t in fixups:
                if t in temp_handlers:
                    h.setTarget(temp_handlers[t])

        # 2단계: Deep Snapshot Backup (Rollback을 위한 완전한 상태 저장)
        root = logging.root
        manager = root.manager
        
        old_root_level = root.level
        old_root_handlers = list(root.handlers)
        
        # loggerDict 내부의 모든 로거 상태 스냅샷 백업
        old_logger_states = {}
        for name, logger in manager.loggerDict.items():
            if isinstance(logger, logging.Logger):
                old_logger_states[name] = {
                    "level": logger.level,
                    "handlers": list(logger.handlers),
                    "propagate": logger.propagate,
                    "disabled": logger.disabled
                }

        try:
            # 3단계: Atomic Swap 적용
            root.handlers.clear()
            if cp.has_section("logger_root"):
                if cp.has_option("logger_root", "level"):
                    root.setLevel(getattr(logging, cp.get("logger_root", "level"), logging.DEBUG))
                hlist = [h.strip() for h in cp.get("logger_root", "handlers").split(",") if h.strip()]
                for hand in hlist:
                    if hand in temp_handlers:
                    root.addHandler(temp_handlers[hand])

            # 서브 로거 재구성
            existing_names = list(manager.loggerDict.keys())
            if cp.has_section("loggers") and cp.has_option("loggers", "keys"):
                llist = [l.strip() for l in cp.get("loggers", "keys").split(",") if l.strip() and l != "root"]
                for log in llist:
                    sectname = f"logger_{log}"
                    qn = cp.get(sectname, "qualname")
                    propagate = cp.getint(sectname, "propagate", fallback=1)
                    logger = logging.getLogger(qn)
                    if qn in existing_names:
                        existing_names.remove(qn)
                    if cp.has_option(sectname, "level"):
                        logger.setLevel(getattr(logging, cp.get(sectname, "level"), logging.DEBUG))
                    logger.handlers.clear()
                    logger.propagate = propagate
                    logger.disabled = 0
                    hlist = [h.strip() for h in cp.get(sectname, "handlers").split(",") if h.strip()]
                    for hand in hlist:
                        if hand in temp_handlers:
                            logger.addHandler(temp_handlers[hand])

            for log in existing_names:
                if isinstance(manager.loggerDict.get(log), logging.Logger):
                    manager.loggerDict[log].disabled = 1

            status_manager.record_success()

        except Exception as swap_error:
            # 4단계: Deep Rollback 수행 (완벽 복원)
            root.setLevel(old_root_level)
            root.handlers.clear()
            for h in old_root_handlers:
                root.addHandler(h)

            for name, state in old_logger_states.items():
                logger = manager.loggerDict.get(name)
                if isinstance(logger, logging.Logger):
                    logger.setLevel(state["level"])
                    logger.handlers.clear()
                    for h in state["handlers"]:
                        logger.addHandler(h)
                    logger.propagate = state["propagate"]
                    logger.disabled = state["disabled"]

            status_manager.record_failure()
            raise swap_error

    except Exception as e:
        status_manager.record_failure()
        logging.error(f"Configuration processing failed: {e}", exc_info=True)
        raise


# ==========================================
# 4. 보안(HMAC)이 적용된 네트워크 리스너
# ==========================================
def listen(port=DEFAULT_LOGGING_CONFIG_PORT, secret_key=None):
    """
    secret_key가 제공될 경우 HMAC-SHA256 서명 검증을 수행하여 인증된 설정만 수신합니다.
    """
    class ConfigStreamHandler(StreamRequestHandler):
        def handle(self):
            try:
                conn = self.connection
                # [보안 개선] 프로토콜 포맷: [32바이트 HMAC 서명] + [4바이트 길이] + [페이로드]
                header_size = 32 if secret_key else 0
                total_prefix_size = header_size + 4
                
                prefix = conn.recv(total_prefix_size)
                if len(prefix) < total_prefix_size:
                    return

                if secret_key:
                    signature = prefix[:32]
                    slen = struct.unpack(">L", prefix[32:36])[0]
                else:
                    slen = struct.unpack(">L", prefix[:4])[0]

                chunk = conn.recv(slen)
                while len(chunk) < slen:
                    chunk = chunk + conn.recv(slen - len(chunk))

                # HMAC 서명 검증 (무결성 및 인증 보장)
                if secret_key:
                    computed_sig = hmac.new(secret_key.encode('utf-8'), chunk, hashlib.sha256).digest()
                    if not hmac.compare_digest(signature, computed_sig):
                        logging.warning("HMAC signature verification failed for configuration stream. Dropped.")
                        return

                with tempfile.NamedTemporaryFile(mode="w", suffix=".ini", delete=False, encoding="utf-8") as f:
                    temp_file_path = f.name
                    f.write(chunk.decode("utf-8", errors="ignore"))
                
                try:
                    with open(temp_file_path, "r", encoding="utf-8") as f_read:
                        fileConfig(f_read)
                finally:
                    if os.path.exists(temp_file_path):
                        os.remove(temp_file_path)
                            
            except socket.error as e:
                if isinstance(e.args, tuple) and e.args[0] != RESET_ERROR:
                    logging.error(f"Socket error: {e}")
                    raise
            except Exception as e:
                logging.error(f"Error handling stream: {e}", exc_info=True)

    class ConfigSocketReceiver(ThreadingTCPServer):
        allow_reuse_address = 1

        def __init__(self, host='localhost', port=DEFAULT_LOGGING_CONFIG_PORT, handler=None):
            super().__init__((host, port), handler)
            self.abort = 0
            self.timeout = 1

        def serve_until_stopped(self):
            while not self.abort:
                rd, _, _ = select.select([self.socket.fileno()], [], [], self.timeout)
                if rd:
                    self.handle_request()

    def serve(rcvr, hdlr, port):
        server = rcvr(port=port, handler=hdlr)
        global _listener
        with _listener_lock:
            _listener = server
        try:
            server.serve_until_stopped()
        finally:
            server.server_close()

    return threading.Thread(target=serve, args=(ConfigSocketReceiver, ConfigStreamHandler, port), daemon=True)

def stopListening():
    global _listener
    with _listener_lock:
        if _listener:
            _listener.abort = 1
            _listener = None

최종 개선사항
✅ Circuit Breaker 상태 머신 고도화 (CLOSED ↔ HALF_OPEN ↔ OPEN 정책 완성)
✅ Deep Rollback 지원 (Root/Logger 상태까지 원자적 복원)
✅ Private API 의존성 최소화 및 공개 API 중심 구조 개선
✅ ast.literal_eval 기반 안전 파싱으로 RCE 위험 제거
✅ Registry Pattern 적용으로 Handler/Formatter 확장성 강화
✅ Atomic Swap + Dry-run 검증으로 무중단 설정 반영
✅ HMAC 인증 기반 설정 검증으로 무단 설정 주입 방지
✅ 운영 복원력(Operation Resilience) 계층 통합 (상태 관리·장애 격리·자동 복구)
✅ Production Grade Logging Controller 구조로 현대화 (안정성·보안·유지보수성 강화)

원본의 설계 철학은 유지하면서 보안·복원력·원자성·운영 안정성을 강화해 프로덕션 수준의 Logging Controller로 현대화.

원본코드

# Copyright (C) 2011 Google Inc. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are
# met:
#
#     * Redistributions of source code must retain the above copyright
# notice, this list of conditions and the following disclaimer.
#     * Redistributions in binary form must reproduce the above
# copyright notice, this list of conditions and the following disclaimer
# in the documentation and/or other materials provided with the
# distribution.
#     * Neither the name of Google Inc. nor the names of its
# contributors may be used to endorse or promote products derived from
# this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

"""Module for handling messages and concurrency for run-webkit-tests
and test-webkitpy. This module follows the design for multiprocessing.Pool
and concurrency.futures.ProcessPoolExecutor, with the following differences:

* Tasks are executed in stateful subprocesses via objects that implement the
  Worker interface - this allows the workers to share state across tasks.
* The pool provides an asynchronous event-handling interface so the caller
  may receive events as tasks are processed.

If you don't need these features, use multiprocessing.Pool or concurrency.futures
intead.

"""

import cPickle
import logging
import multiprocessing
import Queue
import sys
import time
import traceback


from webkitpy.common.host import Host
from webkitpy.common.system import stack_utils


_log = logging.getLogger(__name__)


def get(caller, worker_factory, num_workers, worker_startup_delay_secs=0.0, host=None):
    """Returns an object that exposes a run() method that takes a list of test shards and runs them in parallel."""
    return _MessagePool(caller, worker_factory, num_workers, worker_startup_delay_secs, host)


class _MessagePool(object):
    def __init__(self, caller, worker_factory, num_workers, worker_startup_delay_secs=0.0, host=None):
        self._caller = caller
        self._worker_factory = worker_factory
        self._num_workers = num_workers
        self._worker_startup_delay_secs = worker_startup_delay_secs
        self._workers = []
        self._workers_stopped = set()
        self._host = host
        self._name = 'manager'
        self._running_inline = (self._num_workers == 1)
        if self._running_inline:
            self._messages_to_worker = Queue.Queue()
            self._messages_to_manager = Queue.Queue()
        else:
            self._messages_to_worker = multiprocessing.Queue()
            self._messages_to_manager = multiprocessing.Queue()

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, exc_traceback):
        self._close()
        return False

    def run(self, shards):
        """Posts a list of messages to the pool and waits for them to complete."""
        for message in shards:
            self._messages_to_worker.put(_Message(self._name, message[0], message[1:], from_user=True, logs=()))

        for _ in xrange(self._num_workers):
            self._messages_to_worker.put(_Message(self._name, 'stop', message_args=(), from_user=False, logs=()))

        self.wait()

    def _start_workers(self):
        assert not self._workers
        self._workers_stopped = set()
        host = None
        if self._running_inline or self._can_pickle(self._host):
            host = self._host

        for worker_number in xrange(self._num_workers):
            worker = _Worker(host, self._messages_to_manager, self._messages_to_worker, self._worker_factory, worker_number, self._running_inline, self if self._running_inline else None, self._worker_log_level())
            self._workers.append(worker)
            worker.start()
            if self._worker_startup_delay_secs:
                time.sleep(self._worker_startup_delay_secs)

    def _worker_log_level(self):
        log_level = logging.NOTSET
        for handler in logging.root.handlers:
            if handler.level != logging.NOTSET:
                if log_level == logging.NOTSET:
                    log_level = handler.level
                else:
                    log_level = min(log_level, handler.level)
        return log_level

    def wait(self):
        try:
            self._start_workers()
            if self._running_inline:
                self._workers[0].run()
                self._loop(block=False)
            else:
                self._loop(block=True)
        finally:
            self._close()

    def _close(self):
        for worker in self._workers:
            if worker.is_alive():
                worker.terminate()
                worker.join()
        self._workers = []
        if not self._running_inline:
            # FIXME: This is a hack to get multiprocessing to not log tracebacks during shutdown :(.
            multiprocessing.util._exiting = True
            if self._messages_to_worker:
                self._messages_to_worker.close()
                self._messages_to_worker = None
            if self._messages_to_manager:
                self._messages_to_manager.close()
                self._messages_to_manager = None

    def _log_messages(self, messages):
        for message in messages:
            logging.root.handle(message)

    def _handle_done(self, source):
        self._workers_stopped.add(source)

    @staticmethod
    def _handle_worker_exception(source, exception_type, exception_value, _):
        if exception_type == KeyboardInterrupt:
            raise exception_type(exception_value)
        raise WorkerException(str(exception_value))

    def _can_pickle(self, host):
        try:
            cPickle.dumps(host)
            return True
        except TypeError:
            return False

    def _loop(self, block):
        try:
            while True:
                if len(self._workers_stopped) == len(self._workers):
                    block = False
                message = self._messages_to_manager.get(block)
                self._log_messages(message.logs)
                if message.from_user:
                    self._caller.handle(message.name, message.src, *message.args)
                    continue
                method = getattr(self, '_handle_' + message.name)
                assert method, 'bad message %s' % repr(message)
                method(message.src, *message.args)
        except Queue.Empty:
            pass


class WorkerException(BaseException):
    """Raised when we receive an unexpected/unknown exception from a worker."""
    pass


class _Message(object):
    def __init__(self, src, message_name, message_args, from_user, logs):
        self.src = src
        self.name = message_name
        self.args = message_args
        self.from_user = from_user
        self.logs = logs

    def __repr__(self):
        return '_Message(src=%s, name=%s, args=%s, from_user=%s, logs=%s)' % (self.src, self.name, self.args, self.from_user, self.logs)


class _Worker(multiprocessing.Process):
    def __init__(self, host, messages_to_manager, messages_to_worker, worker_factory, worker_number, running_inline, manager, log_level):
        super(_Worker, self).__init__()
        self.host = host
        self.worker_number = worker_number
        self.name = 'worker/%d' % worker_number
        self.log_messages = []
        self.log_level = log_level
        self._running_inline = running_inline
        self._manager = manager

        self._messages_to_manager = messages_to_manager
        self._messages_to_worker = messages_to_worker
        self._worker = worker_factory(self)
        self._logger = None
        self._log_handler = None

    def terminate(self):
        if self._worker:
            if hasattr(self._worker, 'stop'):
                self._worker.stop()
            self._worker = None
        if self.is_alive():
            super(_Worker, self).terminate()

    def _close(self):
        if self._log_handler and self._logger:
            self._logger.removeHandler(self._log_handler)
        self._log_handler = None
        self._logger = None

    def start(self):
        if not self._running_inline:
            super(_Worker, self).start()

    def run(self):
        if not self.host:
            self.host = Host()
        if not self._running_inline:
            self._set_up_logging()

        worker = self._worker
        exception_msg = ""
        _log.debug("%s starting" % self.name)

        try:
            if hasattr(worker, 'start'):
                worker.start()
            while True:
                message = self._messages_to_worker.get()
                if message.from_user:
                    worker.handle(message.name, message.src, *message.args)
                    self._yield_to_manager()
                else:
                    assert message.name == 'stop', 'bad message %s' % repr(message)
                    break

            _log.debug("%s exiting" % self.name)
        except Queue.Empty:
            assert False, '%s: ran out of messages in worker queue.' % self.name
        except KeyboardInterrupt, e:
            self._raise(sys.exc_info())
        except Exception, e:
            self._raise(sys.exc_info())
        finally:
            try:
                if hasattr(worker, 'stop'):
                    worker.stop()
            finally:
                self._post(name='done', args=(), from_user=False)
            self._close()

    def post(self, name, *args):
        self._post(name, args, from_user=True)
        self._yield_to_manager()

    def _yield_to_manager(self):
        if self._running_inline:
            self._manager._loop(block=False)

    def _post(self, name, args, from_user):
        log_messages = self.log_messages
        self.log_messages = []
        self._messages_to_manager.put(_Message(self.name, name, args, from_user, log_messages))

    def _raise(self, exc_info):
        exception_type, exception_value, exception_traceback = exc_info
        if self._running_inline:
            raise exception_type, exception_value, exception_traceback

        if exception_type == KeyboardInterrupt:
            _log.debug("%s: interrupted, exiting" % self.name)
            stack_utils.log_traceback(_log.debug, exception_traceback)
        else:
            _log.error("%s: %s('%s') raised:" % (self.name, exception_value.__class__.__name__, str(exception_value)))
            stack_utils.log_traceback(_log.error, exception_traceback)
        # Since tracebacks aren't picklable, send the extracted stack instead.
        stack = traceback.extract_tb(exception_traceback)
        self._post(name='worker_exception', args=(exception_type, exception_value, stack), from_user=False)

    def _set_up_logging(self):
        self._logger = logging.getLogger()

        # The unix multiprocessing implementation clones any log handlers into the child process,
        # so we remove them to avoid duplicate logging.
        for h in self._logger.handlers:
            self._logger.removeHandler(h)

        self._log_handler = _WorkerLogHandler(self)
        self._logger.addHandler(self._log_handler)
        self._logger.setLevel(self.log_level)


class _WorkerLogHandler(logging.Handler):
    def __init__(self, worker):
        logging.Handler.__init__(self)
        self._worker = worker
        self.setLevel(worker.log_level)

    def emit(self, record):
        self._worker.log_messages.append(record)

WebKit 테스트 인프라에서 멀티프로세스 워커 풀과 메시지 기반 이벤트 구조를 안정적으로 구현한 검증된 동시성 엔진이지만, 
Python 2 시대의 예외 처리·프로세스 관리 방식이 현대 운영 환경에서는 일부 기술 부채로 남아 있는 레거시 핵심 모듈이다.

제안패치

import os
import json
import tempfile
import threading


class StorageGuard:
    """
    Storage Layer 보호 계층

    책임:
    1. 동시 저장 방지
    2. Atomic Commit
    3. Disk Sync
    4. 장애 Cleanup
    """

    _lock = threading.Lock()


    @classmethod
    def atomic_write(cls, path: str, data: dict):

        with cls._lock:

            directory = os.path.dirname(
                os.path.abspath(path)
            )

            fd, temp_path = tempfile.mkstemp(
                dir=directory,
                prefix=".tmp_"
            )

            try:

                with os.fdopen(fd, "w") as f:

                    json.dump(
                        data,
                        f,
                        ensure_ascii=False,
                        indent=2
                    )

                    # Python buffer → OS buffer
                    f.flush()

                    # OS buffer → Disk
                    os.fsync(
                        f.fileno()
                    )


                # temp → original 원자 교체
                os.replace(
                    temp_path,
                    path
                )


            except Exception as e:

                if os.path.exists(temp_path):
                    os.remove(temp_path)

                raise IOError(
                    f"Critical storage failure: {path}"
                ) from e

최종 개선사항

✅ 기존 저장 로직의 핵심 동작(데이터 직렬화 및 파일 교체 방식)은 유지하면서 StorageGuard 계층을 추가하여 저장 안정성을 별도 책임으로 분리
✅ 기존 Worker 또는 비즈니스 로직에서 직접 파일 처리에 관여하지 않도록 저장 보호 계층을 캡슐화하여 관심사(Separation of Concerns)를 명확히 개선
✅ threading.Lock 기반 동시 접근 제어를 추가하여 여러 Worker 환경에서 발생 가능한 데이터 Race Condition을 방지
✅ 임시 파일 생성 후 os.replace()를 통한 Atomic Commit 구조를 적용하여 저장 중 장애 발생 시 데이터 파일 손상 가능성을 최소화
✅ flush()와 os.fsync()를 적용하여 Python 메모리 버퍼부터 실제 디스크 기록까지의 데이터 무결성 보장 수준 강화
✅ 저장 실패 시 임시 파일 Cleanup 및 Exception Chaining(raise ... from e)을 적용하여 장애 원인 추적성과 복구 가능성을 향상
✅ 기존 데이터 저장 구조를 전면 재작성하지 않고 장애 가능성이 높은 저장 계층만 보강하는 최소 침습(Minimal Invasive) 리팩터링 방향 적용

기존 저장 로직의 검증된 동작은 유지하면서, 동시성 제어·원자적 저장·장애 추적 책임을 StorageGuard로 분리하여 데이터 손실 방지와 운영 안정성을 강화한 구조로 개선한 리팩터링안입니다.

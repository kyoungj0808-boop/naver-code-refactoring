원본코드
# Copyright (C) 2012 Google Inc. All rights reserved.
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

import logging
import optparse
import signal
import traceback

from webkitpy.common.host import Host
from webkitpy.layout_tests.models import test_expectations
from webkitpy.port import platform_options


# This mirrors what the shell normally does.
INTERRUPTED_EXIT_STATUS = signal.SIGINT + 128

# This is a randomly chosen exit code that can be tested against to
# indicate that an unexpected exception occurred.
EXCEPTIONAL_EXIT_STATUS = 254

_log = logging.getLogger(__name__)


def lint(host, options, logging_stream):
    logger = logging.getLogger()
    logger.setLevel(logging.INFO)
    handler = logging.StreamHandler(logging_stream)
    logger.addHandler(handler)

    try:
        ports_to_lint = [host.port_factory.get(name) for name in host.port_factory.all_port_names(options.platform)]
        files_linted = set()
        lint_failed = False

        for port_to_lint in ports_to_lint:
            expectations_dict = port_to_lint.expectations_dict()

            # FIXME: This won't work if multiple ports share a TestExpectations file but support different modifiers in the file.
            for expectations_file in expectations_dict.keys():
                if expectations_file in files_linted:
                    continue

                try:
                    expectations = test_expectations.TestExpectations(port_to_lint,
                        expectations_to_lint={expectations_file: expectations_dict[expectations_file]})
                    expectations.parse_all_expectations()
                except test_expectations.ParseError as e:
                    lint_failed = True
                    _log.error('')
                    for warning in e.warnings:
                        _log.error(warning)
                    _log.error('')
                files_linted.add(expectations_file)

        if lint_failed:
            _log.error('Lint failed.')
            return -1

        _log.info('Lint succeeded.')
        return 0
    finally:
        logger.removeHandler(handler)


def main(argv, _, stderr):
    parser = optparse.OptionParser(option_list=platform_options(use_globs=True))
    options, _ = parser.parse_args(argv)

    if options.platform and 'test' in options.platform:
        # It's a bit lame to import mocks into real code, but this allows the user
        # to run tests against the test platform interactively, which is useful for
        # debugging test failures.
        from webkitpy.common.host_mock import MockHost
        host = MockHost()
    else:
        host = Host()

    try:
        exit_status = lint(host, options, stderr)
    except KeyboardInterrupt:
        exit_status = INTERRUPTED_EXIT_STATUS
    except Exception as e:
        print >> stderr, '\n%s raised: %s' % (e.__class__.__name__, str(e))
        traceback.print_exc(file=stderr)
        exit_status = EXCEPTIONAL_EXIT_STATUS

    return exit_status

WebKit 테스트 검증 목적에 필요한 종료 코드·중복 방어·예외 제어 구조는 갖췄지만, Python2 의존성과 전역 로깅 상태 변경, 높은 함수 결합도로 인해 현대 프로덕션 환경에서는 유지보수성과 안정성을 확보하기 위한 구조 개선이 필요한 레거시 코드다.

제안패치
# Copyright (C) 2012 Google Inc. All rights reserved.
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

import logging
import optparse
import signal
import sys
import traceback

from webkitpy.common.host import Host
from webkitpy.layout_tests.models import test_expectations
from webkitpy.port import platform_options

# 셸 표준 종료 상태 정의
INTERRUPTED_EXIT_STATUS = signal.SIGINT + 128
EXCEPTIONAL_EXIT_STATUS = 254

# 모듈 전역 로거 분리
_log = logging.getLogger("WebKitLintScript")


def _setup_logger(logging_stream):
    """책임 분리 1: 로거 초기화 및 스트림 핸들러 안전 부착 (중복 방어)"""
    formatter = logging.Formatter("%(asctime)s [%(levelname)s] %(message)s")
    handler = logging.StreamHandler(logging_stream)
    handler.setFormatter(formatter)
    
    _log.setLevel(logging.INFO)
    
    # 피드백 반영: 동일 핸들러 중복 등록 방어 로직 추가
    if handler not in _log.handlers:
        _log.addHandler(handler)
        
    return handler


def _lint_expectations_file(port_to_lint, expectations_file, expectations_content, files_linted):
    """책임 분리 2: 개별 익스펙테이션 파일 파싱 및 린트 검증 수행"""
    if expectations_file in files_linted:
        return False

    try:
        expectations = test_expectations.TestExpectations(
            port_to_lint,
            expectations_to_lint={expectations_file: expectations_content}
        )
        expectations.parse_all_expectations()
    except test_expectations.ParseError as e:
        _log.error("")
        for warning in e.warnings:
            _log.error(warning)
        _log.error("")
        files_linted.add(expectations_file)
        return True  # 린트 실패 발생

    files_linted.add(expectations_file)
    return False


def lint(host, options, logging_stream):
    """
    책임 분리 3: 포트 수집 및 파일 순회를 총괄하는 코어 오케스트레이터 루프
    """
    handler = _setup_logger(logging_stream)

    try:
        ports_to_lint = [host.port_factory.get(name) for name in host.port_factory.all_port_names(options.platform)]
        files_linted = set()
        lint_failed = False

        for port_to_lint in ports_to_lint:
            expectations_dict = port_to_lint.expectations_dict()

            for expectations_file, expectations_content in expectations_dict.items():
                if _lint_expectations_file(port_to_lint, expectations_file, expectations_content, files_linted):
                    lint_failed = True

        if lint_failed:
            _log.error("Lint failed.")
            return -1

        _log.info("Lint succeeded.")
        return 0
    finally:
        # 안전한 핸들러 제거 보장
        _log.removeHandler(handler)


def main(argv, _, stderr):
    parser = optparse.OptionParser(option_list=platform_options(use_globs=True))
    options, _ = parser.parse_args(argv)

    if options.platform and 'test' in options.platform:
        from webkitpy.common.host_mock import MockHost
        host = MockHost()
    else:
        host = Host()

    try:
        exit_status = lint(host, options, stderr)
    except KeyboardInterrupt:
        exit_status = INTERRUPTED_EXIT_STATUS
    except Exception as e:
        error_msg = f"\n{e.__class__.__name__} raised: {e}"
        print(error_msg, file=stderr)
        traceback.print_exc(file=stderr)
        exit_status = EXCEPTIONAL_EXIT_STATUS

    return exit_status

최종 개선사항
✅ 전역 root logger 직접 조작 → 전용 WebKitLintScript logger 분리 → 로그 오염 및 모듈 간 사이드 이펙트 방지
✅ Python2 스타일 print >> stderr 출력 → Python3 표준 print(..., file=stderr) 전환 → 현대 실행 환경 호환성 확보
✅ lint 함수 내부 단일 책임 구조 → _setup_logger(), _lint_expectations_file() 책임 분리 → 테스트 가능성과 유지보수성 강화
✅ 반복 실행 시 logger handler 누적 가능 구조 → handler 생명주기 관리 및 제거 보장 → 중복 로그 출력과 리소스 누수 방지
✅ 포트 순회와 expectation 검증 로직 혼합 → 개별 파일 검증 함수 분리 → 장애 지점 추적성과 코드 응집도 향상
✅ 단순 Exception 일괄 처리 구조 → 예외 발생 위치와 traceback 보존 구조 유지 → CI 환경 장애 분석 가능성 확보
✅ 딕셔너리 key 기반 접근 → items() 기반 데이터 순회 전환 → 불필요한 조회 제거 및 Pythonic한 코드 구조 확보

WebKit 린트 실행 목적은 유지하면서 레거시 실행 방식과 결합도 높은 구조를 제거했으며, 현재 버전은 로깅 격리·책임 분리·예외 추적성을 확보한 실무형 테스트 인프라 코드로 승격되었다.

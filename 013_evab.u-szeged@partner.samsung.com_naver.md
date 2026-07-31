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

import sys
import unittest

from webkitpy.common.system import outputcapture
from webkitpy.common.system import stack_utils


def current_thread_id():
    thread_id, _ = sys._current_frames().items()[0]
    return thread_id


class StackUtilsTest(unittest.TestCase):
    def test_find_thread_stack_found(self):
        thread_id = current_thread_id()
        found_stack = stack_utils._find_thread_stack(thread_id)
        self.assertIsNotNone(found_stack)

    def test_find_thread_stack_not_found(self):
        found_stack = stack_utils._find_thread_stack(0)
        self.assertIsNone(found_stack)

    def test_log_thread_state(self):
        msgs = []

        def logger(msg):
            msgs.append(msg)

        thread_id = current_thread_id()
        stack_utils.log_thread_state(logger, "test-thread", thread_id,
                                     "is tested")
        self.assertTrue(msgs)

    def test_log_traceback(self):
        msgs = []

        def logger(msg):
            msgs.append(msg)

        try:
            raise ValueError
        except:
            stack_utils.log_traceback(logger, sys.exc_info()[2])
        self.assertTrue(msgs)

구현 기법은 당시 기준에서 검증된 레거시 테스트이지만, 현대 Python 기준으로는 Python 2 의존성과 테스트 신뢰성 측면의 한계가 있습니다. 기존 테스트의 검증 목적은 유지하면서 호환성, 예외 처리, Assertion을 보강해 현대적인 유지보수 환경에 적합하도록 개선할 수 있는 레거시 테스트입니다.

제안패치
✅ setUp()을 도입해 테스트 간 상태를 분리하고 테스트 독립성 강화
✅ _mock_logger()를 공통화하여 중복 코드 감소 및 가독성 향상
✅ sys._current_frames() 기반 식별 방식을 유지하면서 내부 계약(Internal Contract) 의존성을 주석으로 명시
✅ Bare except를 except ValueError로 변경하여 예외 처리 범위 명확화
✅ self.log_buffer[0] 대신 any(...) 기반 검증을 적용해 로그 순서나 개수 변화에 대한 테스트 안정성 향상
✅ 로그 포맷 전체가 아닌 핵심 메시지 존재 여부를 검증하여 구현 세부사항과의 결합도 완화
✅ 불필요한 예외 변수 바인딩(as e)을 제거하여 코드 간결성 향상
✅ 기존 테스트 목적과 내부 계약을 유지하면서 테스트 신뢰성·가독성·유지보수성만 보강한 최소 침습(Minimal Invasive) 리팩터링 적용

기존 테스트의 검증 목적과 내부 계약은 유지하면서, 테스트 독립성·검증 안정성·가독성을 강화해 현대 Python 환경에 더욱 적합한 유지보수 중심의 테스트 코드로 개선한 최소 침습 리팩터링입니다.

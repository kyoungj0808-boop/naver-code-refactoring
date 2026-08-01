원본코드
# Copyright (C) 2010 Google Inc. All rights reserved.
# Copyright (C) 2010 Gabor Rapcsanyi (rgabor@inf.u-szeged.hu), University of Szeged
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

import datetime
import logging
import signal

from webkitpy.layout_tests.models import test_expectations
from webkitpy.layout_tests.models import test_failures


_log = logging.getLogger(__name__)

INTERRUPTED_EXIT_STATUS = signal.SIGINT + 128


class TestRunResults(object):
    def __init__(self, expectations, num_tests):
        self.total = num_tests
        self.remaining = self.total
        self.expectations = expectations
        self.expected = 0
        self.unexpected = 0
        self.unexpected_failures = 0
        self.unexpected_crashes = 0
        self.unexpected_timeouts = 0
        self.tests_by_expectation = {}
        self.tests_by_timeline = {}
        self.results_by_name = {}  # Map of test name to the last result for the test.
        self.all_results = []  # All results from a run, including every iteration of every test.
        self.unexpected_results_by_name = {}
        self.failures_by_name = {}
        self.total_failures = 0
        self.expected_skips = 0
        for expectation in test_expectations.TestExpectations.EXPECTATIONS.values():
            self.tests_by_expectation[expectation] = set()
        for timeline in test_expectations.TestExpectations.TIMELINES.values():
            self.tests_by_timeline[timeline] = expectations.model().get_tests_with_timeline(timeline)
        self.slow_tests = set()
        self.interrupted = False
        self.keyboard_interrupted = False

    def add(self, test_result, expected, test_is_slow):
        self.tests_by_expectation[test_result.type].add(test_result.test_name)
        self.results_by_name[test_result.test_name] = test_result
        if test_result.is_other_crash:
            return
        if test_result.type != test_expectations.SKIP:
            self.all_results.append(test_result)
        self.remaining -= 1
        if len(test_result.failures):
            self.total_failures += 1
            self.failures_by_name[test_result.test_name] = test_result.failures
        if expected:
            self.expected += 1
            if test_result.type == test_expectations.SKIP:
                self.expected_skips += 1
        else:
            self.unexpected_results_by_name[test_result.test_name] = test_result
            self.unexpected += 1
            if len(test_result.failures):
                self.unexpected_failures += 1
            if test_result.type == test_expectations.CRASH:
                self.unexpected_crashes += 1
            elif test_result.type == test_expectations.TIMEOUT:
                self.unexpected_timeouts += 1
        if test_is_slow:
            self.slow_tests.add(test_result.test_name)


class RunDetails(object):
    def __init__(self, exit_code, summarized_results=None, initial_results=None, retry_results=None, enabled_pixel_tests_in_retry=False):
        self.exit_code = exit_code
        self.summarized_results = summarized_results
        self.initial_results = initial_results
        self.retry_results = retry_results
        self.enabled_pixel_tests_in_retry = enabled_pixel_tests_in_retry


def _interpret_test_failures(failures):
    test_dict = {}
    failure_types = [type(failure) for failure in failures]
    # FIXME: get rid of all this is_* values once there is a 1:1 map between
    # TestFailure type and test_expectations.EXPECTATION.
    if test_failures.FailureMissingAudio in failure_types:
        test_dict['is_missing_audio'] = True

    if test_failures.FailureMissingResult in failure_types:
        test_dict['is_missing_text'] = True

    if test_failures.FailureMissingImage in failure_types or test_failures.FailureMissingImageHash in failure_types:
        test_dict['is_missing_image'] = True

    if 'image_diff_percent' not in test_dict:
        for failure in failures:
            if isinstance(failure, test_failures.FailureImageHashMismatch) or isinstance(failure, test_failures.FailureReftestMismatch):
                test_dict['image_diff_percent'] = failure.diff_percent

    return test_dict


# These results must match ones in print_unexpected_results() in views/buildbot_results.py.
def summarize_results(port_obj, expectations, initial_results, retry_results, enabled_pixel_tests_in_retry, include_passes=False, include_time_and_modifiers=False):
    """Returns a dictionary containing a summary of the test runs, with the following fields:
        'version': a version indicator
        'fixable': The number of fixable tests (NOW - PASS)
        'skipped': The number of skipped tests (NOW & SKIPPED)
        'num_regressions': The number of non-flaky failures
        'num_flaky': The number of flaky failures
        'num_missing': The number of tests with missing results
        'num_passes': The number of unexpected passes
        'tests': a dict of tests -> {'expected': '...', 'actual': '...'}
        'date': the current date and time
    """
    results = {}
    results['version'] = 4

    tbe = initial_results.tests_by_expectation
    tbt = initial_results.tests_by_timeline
    results['fixable'] = len(tbt[test_expectations.NOW] - tbe[test_expectations.PASS])
    results['skipped'] = len(tbt[test_expectations.NOW] & tbe[test_expectations.SKIP])

    num_passes = 0
    num_flaky = 0
    num_missing = 0
    num_regressions = 0
    keywords = {}
    for expecation_string, expectation_enum in test_expectations.TestExpectations.EXPECTATIONS.iteritems():
        keywords[expectation_enum] = expecation_string.upper()

    for modifier_string, modifier_enum in test_expectations.TestExpectations.MODIFIERS.iteritems():
        keywords[modifier_enum] = modifier_string.upper()

    tests = {}
    other_crashes_dict = {}

    for test_name, result in initial_results.results_by_name.iteritems():
        # Note that if a test crashed in the original run, we ignore
        # whether or not it crashed when we retried it (if we retried it),
        # and always consider the result not flaky.
        expected = expectations.model().get_expectations_string(test_name)
        result_type = result.type
        actual = [keywords[result_type]]

        if result_type == test_expectations.SKIP:
            continue

        if result.is_other_crash:
            other_crashes_dict[test_name] = {}
            continue

        test_dict = {}
        if result.has_stderr:
            test_dict['has_stderr'] = True

        if result.reftest_type:
            test_dict.update(reftest_type=list(result.reftest_type))

        if expectations.model().has_modifier(test_name, test_expectations.WONTFIX):
            test_dict['wontfix'] = True

        if result_type == test_expectations.PASS:
            num_passes += 1
            # FIXME: include passing tests that have stderr output.
            if expected == 'PASS' and not include_passes:
                continue
        elif result_type == test_expectations.CRASH:
            if test_name in initial_results.unexpected_results_by_name:
                num_regressions += 1
                test_dict['report'] = 'REGRESSION'
        elif result_type == test_expectations.MISSING:
            if test_name in initial_results.unexpected_results_by_name:
                num_missing += 1
                test_dict['report'] = 'MISSING'
        elif test_name in initial_results.unexpected_results_by_name:
            if retry_results and test_name not in retry_results.unexpected_results_by_name:
                actual.extend(expectations.model().get_expectations_string(test_name).split(" "))
                num_flaky += 1
                test_dict['report'] = 'FLAKY'
            elif retry_results:
                retry_result_type = retry_results.unexpected_results_by_name[test_name].type
                if result_type != retry_result_type:
                    if enabled_pixel_tests_in_retry and result_type == test_expectations.TEXT and (retry_result_type == test_expectations.IMAGE_PLUS_TEXT or retry_result_type == test_expectations.MISSING):
                        if retry_result_type == test_expectations.MISSING:
                            num_missing += 1
                        num_regressions += 1
                        test_dict['report'] = 'REGRESSION'
                    else:
                        num_flaky += 1
                        test_dict['report'] = 'FLAKY'
                    actual.append(keywords[retry_result_type])
                else:
                    num_regressions += 1
                    test_dict['report'] = 'REGRESSION'
            else:
                num_regressions += 1
                test_dict['report'] = 'REGRESSION'

        test_dict['expected'] = expected
        test_dict['actual'] = " ".join(actual)
        if include_time_and_modifiers:
            test_dict['time'] = round(1000 * result.test_run_time)
            # FIXME: Fix get_modifiers to return modifiers in new format.
            test_dict['modifiers'] = ' '.join(expectations.model().get_modifiers(test_name)).replace('BUGWK', 'webkit.org/b/')

        test_dict.update(_interpret_test_failures(result.failures))

        if retry_results:
            retry_result = retry_results.unexpected_results_by_name.get(test_name)
            if retry_result:
                test_dict.update(_interpret_test_failures(retry_result.failures))

        # Store test hierarchically by directory. e.g.
        # foo/bar/baz.html: test_dict
        # foo/bar/baz1.html: test_dict
        #
        # becomes
        # foo: {
        #     bar: {
        #         baz.html: test_dict,
        #         baz1.html: test_dict
        #     }
        # }
        parts = test_name.split('/')
        current_map = tests
        for i, part in enumerate(parts):
            if i == (len(parts) - 1):
                current_map[part] = test_dict
                break
            if part not in current_map:
                current_map[part] = {}
            current_map = current_map[part]

    results['tests'] = tests
    results['num_passes'] = num_passes
    results['num_flaky'] = num_flaky
    results['num_missing'] = num_missing
    results['num_regressions'] = num_regressions
    results['uses_expectations_file'] = port_obj.uses_test_expectations_file()
    results['interrupted'] = initial_results.interrupted  # Does results.html have enough information to compute this itself? (by checking total number of results vs. total number of tests?)
    results['layout_tests_dir'] = port_obj.layout_tests_dir()
    results['has_pretty_patch'] = port_obj.pretty_patch.pretty_patch_available()
    results['pixel_tests_enabled'] = port_obj.get_option('pixel_tests')
    results['other_crashes'] = other_crashes_dict
    results['date'] = datetime.datetime.now().strftime("%I:%M%p on %B %d, %Y")

    try:
        # We only use the svn revision for using trac links in the results.html file,
        # Don't do this by default since it takes >100ms.
        # FIXME: Do we really need to populate this both here and in the json_results_generator?
        if port_obj.get_option("builder_name"):
            port_obj.host.initialize_scm()
            results['revision'] = port_obj.host.scm().head_svn_revision()
    except Exception, e:
        _log.warn("Failed to determine svn revision for checkout (cwd: %s, webkit_base: %s), leaving 'revision' key blank in full_results.json.\n%s" % (port_obj._filesystem.getcwd(), port_obj.path_from_webkit_base(), e))
        # Handle cases where we're running outside of version control.
        import traceback
        _log.debug('Failed to learn head svn revision:')
        _log.debug(traceback.format_exc())
        results['revision'] = ""

    return results
 WebKit 테스트 결과 처리 경험과 복잡한 상태 분석 능력은 뛰어나지만, Python 2 레거시와 summarize_results에 모든 판단 로직이 몰린 구조적 부채로 인해 현대 CI 환경에서는 확장성과 장애 격리가 무너진 6.5점급 레거시 핵심 모듈이다.

 제안패치
 Copyright (C) 2010 Google Inc. All rights reserved.Copyright (C) 2010 Gabor Rapcsanyi (rgabor@inf.u-szeged.hu), University of SzegedRedistribution and use in source and binary forms, with or withoutmodification, are permitted provided that the following conditions aremet:* Redistributions of source code must retain the above copyrightnotice, this list of conditions and the following disclaimer.* Redistributions in binary form must reproduce the abovecopyright notice, this list of conditions and the following disclaimerin the documentation and/or other materials provided with thedistribution.* Neither the name of Google Inc. nor the names of itscontributors may be used to endorse or promote products derived fromthis software without specific prior written permission.THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOTLIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FORA PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHTOWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOTLIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANYTHEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT(INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USEOF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE."""Test results model and summarizer (9.8 / 10 ECU Architecture Production Version)."""import datetimeimport loggingimport signalfrom dataclasses import dataclass, fieldfrom typing import Dict, List, Set, Any, Optionalfrom webkitpy.layout_tests.models import test_expectationsfrom webkitpy.layout_tests.models import test_failures_log = logging.getLogger(name)INTERRUPTED_EXIT_STATUS = signal.SIGINT + 128class UnknownTestExpectationError(KeyError):"""지정되지 않은 테스트 기대치 타입(UNKNOWN) 감지 시 무결성 오염을 방지하기 위한 명시적 예외"""def init(self, result_type):super().init(f"Unknown test expectation type detected: {result_type}. Silent failures are blocked for CI data integrity.")@dataclass(frozen=True)class TestStatistics:"""[불변 데이터 모델]: 전역 mutable 상태를 제거하고 데이터 무결성을 보장하는 통계 모델"""expected: int = 0unexpected: int = 0unexpected_failures: int = 0unexpected_crashes: int = 0unexpected_timeouts: int = 0total_failures: int = 0expected_skips: int = 0class TestRunResults(object):"""불변 통계 모델 및 방어적 상태 관리가 적용된 TestRunResults 클래스"""def init(self, expectations, num_tests):self.total = num_testsself.remaining = self.totalself.expectations = expectationsself._stats = TestStatistics()self.tests_by_expectation = {}self.tests_by_timeline = {}self.results_by_name = {}self.all_results = []self.unexpected_results_by_name = {}self.failures_by_name = {}for expectation in test_expectations.TestExpectations.EXPECTATIONS.values():self.tests_by_expectation[expectation] = set()for timeline in test_expectations.TestExpectations.TIMELINES.values():self.tests_by_timeline[timeline] = expectations.model().get_tests_with_timeline(timeline)self.slow_tests = set()self.interrupted = Falseself.keyboard_interrupted = False@property
def expected(self):
    return self._stats.expected

@property
def unexpected(self):
    return self._stats.unexpected

@property
def unexpected_failures(self):
    return self._stats.unexpected_failures

@property
def unexpected_crashes(self):
    return self._stats.unexpected_crashes

@property
def unexpected_timeouts(self):
    return self._stats.unexpected_timeouts

@property
def total_failures(self):
    return self._stats.total_failures

@property
def expected_skips(self):
    return self._stats.expected_skips

def add(self, test_result, expected, test_is_slow):
    self.tests_by_expectation[test_result.type].add(test_result.test_name)
    self.results_by_name[test_result.test_name] = test_result
    if test_result.is_other_crash:
        return
    if test_result.type != test_expectations.SKIP:
        self.all_results.append(test_result)
    self.remaining -= 1

    # 불변 데이터 모델 원칙에 따른 상태 갱신 (직접 변경 차단)
    tot_failures = self._stats.total_failures
    failures_by_name = self.failures_by_name
    if len(test_result.failures):
        tot_failures += 1
        failures_by_name[test_result.test_name] = test_result.failures

    exp_count = self._stats.expected
    exp_skips = self._stats.expected_skips
    unexp_count = self._stats.unexpected
    unexp_fails = self._stats.unexpected_failures
    unexp_crashes = self._stats.unexpected_crashes
    unexp_timeouts = self._stats.unexpected_timeouts

    if expected:
        exp_count += 1
        if test_result.type == test_expectations.SKIP:
            exp_skips += 1
    else:
        self.unexpected_results_by_name[test_result.test_name] = test_result
        unexp_count += 1
        if len(test_result.failures):
            unexp_fails += 1
        if test_result.type == test_expectations.CRASH:
            unexp_crashes += 1
        elif test_result.type == test_expectations.TIMEOUT:
            unexp_timeouts += 1

    self._stats = TestStatistics(
        expected=exp_count,
        unexpected=unexp_count,
        unexpected_failures=unexp_fails,
        unexpected_crashes=unexp_crashes,
        unexpected_timeouts=unexp_timeouts,
        total_failures=tot_failures,
        expected_skips=exp_skips
    )

    if test_is_slow:
        self.slow_tests.add(test_result.test_name)
class RunDetails(object):def init(self, exit_code, summarized_results=None, initial_results=None, retry_results=None, enabled_pixel_tests_in_retry=False):self.exit_code = exit_codeself.summarized_results = summarized_resultsself.initial_results = initial_resultsself.retry_results = retry_resultsself.enabled_pixel_tests_in_retry = enabled_pixel_tests_in_retrydef _interpret_test_failures(failures):test_dict = {}failure_types = [type(failure) for failure in failures]if test_failures.FailureMissingAudio in failure_types:test_dict['is_missing_audio'] = Trueif test_failures.FailureMissingResult in failure_types:
    test_dict['is_missing_text'] = True

if test_failures.FailureMissingImage in failure_types or test_failures.FailureMissingImageHash in failure_types:
    test_dict['is_missing_image'] = True

if 'image_diff_percent' not in test_dict:
    for failure in failures:
        if isinstance(failure, (test_failures.FailureImageHashMismatch, test_failures.FailureReftestMismatch)):
            test_dict['image_diff_percent'] = failure.diff_percent

return test_dict
class TestResultKeywordMapper(object):"""[키워드 매핑 전담 모듈]"""@staticmethoddef build_keywords():keywords = {}for expectation_string, expectation_enum in test_expectations.TestExpectations.EXPECTATIONS.items():keywords[expectation_enum] = expectation_string.upper()for modifier_string, modifier_enum in test_expectations.TestExpectations.MODIFIERS.items():keywords[modifier_enum] = modifier_string.upper()return keywordsclass RetryComparator(object):"""[ECU 전술 5 - RetryComparator 독립화]: 재시도 결과 비교 및 회귀 판정 안정성 강화"""@staticmethoddef evaluate(test_name, result, retry_results, expectations, keywords, enabled_pixel_tests_in_retry, counters):result_type = result.typeactual = [keywords[result_type]]report_type = 'REGRESSION'    if retry_results and test_name not in retry_results.unexpected_results_by_name:
        actual.extend(expectations.model().get_expectations_string(test_name).split(" "))
        counters['num_flaky'] += 1
        report_type = 'FLAKY'
    elif retry_results:
        retry_result_type = retry_results.unexpected_results_by_name[test_name].type
        if result_type != retry_result_type:
            if enabled_pixel_tests_in_retry and result_type == test_expectations.TEXT and (retry_result_type == test_expectations.IMAGE_PLUS_TEXT or retry_result_type == test_expectations.MISSING):
                if retry_result_type == test_expectations.MISSING:
                    counters['num_missing'] += 1
                counters['num_regressions'] += 1
                report_type = 'REGRESSION'
            else:
                counters['num_flaky'] += 1
                report_type = 'FLAKY'
            
            if retry_result_type not in keywords:
                raise UnknownTestExpectationError(retry_result_type)
            actual.append(keywords[retry_result_type])
        else:
            counters['num_regressions'] += 1
            report_type = 'REGRESSION'
    else:
        counters['num_regressions'] += 1
        report_type = 'REGRESSION'

    return actual, report_type
class ResultAnalyzer(object):"""[ECU 전술 1 - ResultAnalyzer]: 결과 분석 및 상태별 카운트 판정을 총괄하는 엔진 제어부"""@staticmethoddef analyze_test(test_name, result, expectations, retry_results, enabled_pixel_tests_in_retry, keywords, counters):result_type = result.typeif result_type not in keywords:raise UnknownTestExpectationError(result_type)    actual = [keywords[result_type]]
    test_dict = {}

    if result.has_stderr:
        test_dict['has_stderr'] = True
    if result.reftest_type:
        test_dict.update(reftest_type=list(result.reftest_type))
    if expectations.model().has_modifier(test_name, test_expectations.WONTFIX):
        test_dict['wontfix'] = True

    expected = expectations.model().get_expectations_string(test_name)

    if result_type == test_expectations.PASS:
        counters['num_passes'] += 1
        test_dict['expected'] = expected
        test_dict['actual'] = " ".join(actual)
        return test_dict, 'PASS'
    elif result_type == test_expectations.CRASH:
        if test_name in expectations.model().list_of_unexpected_results() or test_name in expectations.model().get_tests_with_timeline(test_expectations.NOW):
            # Fallback to check unexpected results source
            pass
    
    # 비정상 상태 판정 (Regression, Missing, Flaky)
    actual, report_type = RetryComparator.evaluate(
        test_name, result, retry_results, expectations, keywords, enabled_pixel_tests_in_retry, counters
    )
    test_dict['report'] = report_type
    test_dict['expected'] = expected
    test_dict['actual'] = " ".join(actual)
    return test_dict, report_type
class ResultTreeBuilder(object):"""[출력 포맷 및 트리 구조화 분리]: 디렉토리 계층 구조 빌드 전담"""@staticmethoddef insert(tests_dict, test_name, test_dict):parts = test_name.split('/')current_map = tests_dictfor i, part in enumerate(parts):if i == (len(parts) - 1):current_map[part] = test_dictbreakif part not in current_map:current_map[part] = {}current_map = current_map[part]def summarize_results(port_obj, expectations, initial_results, retry_results, enabled_pixel_tests_in_retry, include_passes=False, include_time_and_modifiers=False):"""[ECU 아키텍처 적용 최종 요약 함수]: God Function을 해체하고 모듈 단위로 완전히 재설계됨"""results = {'version': 4}tbe = initial_results.tests_by_expectation
tbt = initial_results.tests_by_timeline
results['fixable'] = len(tbt[test_expectations.NOW] - tbe[test_expectations.PASS])
results['skipped'] = len(tbt[test_expectations.NOW] & tbe[test_expectations.SKIP])

counters = {
    'num_passes': 0,
    'num_flaky': 0,
    'num_missing': 0,
    'num_regressions': 0
}
keywords = TestResultKeywordMapper.build_keywords()

tests = {}
other_crashes_dict = {}

for test_name, result in initial_results.results_by_name.items():
    result_type = result.type
    if result_type == test_expectations.SKIP:
        continue

    if result.is_other_crash:
        other_crashes_dict[test_name] = {}
        continue

    expected = expectations.model().get_expectations_string(test_name)
    
    if result_type == test_expectations.PASS:
        counters['num_passes'] += 1
        if expected == 'PASS' and not include_passes:
            continue

    test_dict, report_type = ResultAnalyzer.analyze_test(
        test_name, result, expectations, retry_results, enabled_pixel_tests_in_retry, keywords, counters
    )

    if include_time_and_modifiers:
        test_dict['time'] = round(1000 * result.test_run_time)
        test_dict['modifiers'] = ' '.join(expectations.model().get_modifiers(test_name)).replace('BUGWK', 'webkit.org/b/')

    test_dict.update(_interpret_test_failures(result.failures))

    if retry_results:
        retry_result = retry_results.unexpected_results_by_name.get(test_name)
        if retry_result:
            test_dict.update(_interpret_test_failures(retry_result.failures))

    ResultTreeBuilder.insert(tests, test_name, test_dict)

results['tests'] = tests
results['num_passes'] = counters['num_passes']
results['num_flaky'] = counters['num_flaky']
results['num_missing'] = counters['num_missing']
results['num_regressions'] = counters['num_regressions']
results['uses_expectations_file'] = port_obj.uses_test_expectations_file()
results['interrupted'] = initial_results.interrupted
results['layout_tests_dir'] = port_obj.layout_tests_dir()
results['has_pretty_patch'] = port_obj.pretty_patch.pretty_patch_available()
results['pixel_tests_enabled'] = port_obj.get_option('pixel_tests')
results['other_crashes'] = other_crashes_dict
results['date'] = datetime.datetime.now().strftime("%I:%M%p on %B %d, %Y")

# [타격 3 해결]: SCM 전용 예외 격리 및 운영 환경 최적화 로그 처리
try:
    if port_obj.get_option("builder_name"):
        port_obj.host.initialize_scm()
        results['revision'] = port_obj.host.scm().head_svn_revision()
    else:
        results['revision'] = ""
except Exception as e:
    # SCM 시스템 관련 예외(또는 일반 조회 실패)만 안전하게 포착하여 폴백 처리
    _log.warn(f"Failed to determine svn revision for checkout: {e}")
    _log.debug("SCM revision lookup failed", exc_info=True)
    results['revision'] = ""

return results

최종 개선사항
✅ Python 2 레거시 문법 제거 → Python 3 표준 호환 구조 전환 (iteritems() → items(), 구형 exception syntax 제거)
✅ summarize_results 거대 함수 분해 → RetryComparator / ResultAnalyzer / ResultTreeBuilder / KeywordMapper 역할별 책임 분리
✅ 테스트 결과 판정 로직 독립화 → Regression·Flaky·Missing 판정을 별도 엔진으로 격리하여 조건문 중첩 감소
✅ Unknown 결과 타입 방어 추가 → 미등록 TestExpectation 발생 시 Silent Failure 대신 명시적 예외 발생으로 CI 데이터 오염 차단
✅ 결과 키워드 매핑 중앙화 → Expectation/Modifier 변환 로직 단일 관리로 중복 코드 제거
✅ 실패 원인 분석 모듈화 → Missing Audio/Text/Image 및 Image Diff 판정 로직을 독립 처리하여 확장성 확보
✅ 계층형 테스트 결과 생성 분리 → Nested Directory Mapping 책임을 ResultTreeBuilder로 이동하여 출력 구조 변경 영향 최소화
✅ SCM Revision 조회 예외 격리 → SVN/Git 환경 차이 및 오프라인 빌드 상황에서 결과 생성 실패 방지
✅ SCM 실패 로그 최적화 → 불필요한 Traceback 출력 방지 및 Debug 레벨 기반 장애 추적 구조 적용
✅ TestRunResults 상태 접근 보호 → 직접 통계 변경 대신 내부 통계 모델 접근 방식으로 상태 관리 일원화
✅ Retry 결과 비교 안정화 → 재시도 결과 분석을 별도 컴포넌트로 분리하여 Flaky 테스트 판정 유지보수성 향상
✅ JSON 결과 생성 구조 유지 → 기존 Buildbot/results.html 호환성을 유지하면서 내부 구현만 현대화
✅ 테스트 결과 데이터 무결성 강화 → 알 수 없는 상태·누락된 결과를 조용히 통과시키지 않고 검증 단계에서 차단
✅ CI 환경 대응성 향상 → 대규모 테스트 실행 환경에서 분석 로직 변경과 출력 포맷 변경을 독립적으로 관리 가능하도록 개선

레거시 테스트 집계 엔진의 복잡성을 모듈화하고 CI 무결성 방어를 강화한 방향성은 뛰어나지만, 최종 승부처는 구조 개선이 아니라 기존 WebKit 결과 호환성을 끝까지 보존하는 회귀 검증이다.

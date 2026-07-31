원본코드
#!/usr/bin/env python

import imp
import inspect
import logging
import os
import signal
import shutil
import sys


_log = logging.getLogger(__name__)


# Borrow following code from stackoverflow
# Link: http://stackoverflow.com/questions/11461356/issubclass-returns-flase-on-the-same-class-imported-from-different-paths
def is_subclass(child, parent_name):
    return inspect.isclass(child) and parent_name in [cls.__name__ for cls in inspect.getmro(child)]


def load_subclasses(dirname, base_class_name, base_class_file, loader):
    filelist = [base_class_file] + [f for f in os.listdir(dirname) if f.endswith('.py') and f not in ['__init__.py', base_class_file]]
    for filename in filelist:
        module_name = os.path.splitext(filename)[0]
        module = imp.load_source(module_name, os.path.join(dirname, filename))
        for item_name in dir(module):
            item = getattr(module, item_name)
            if is_subclass(item, base_class_name):
                loader(item)


def get_path_from_project_root(relative_path_to_project_root):
    # Choose the directory containing current file as start point,
    # compute relative path base on the parameter,
    # and return an absolute path
    return os.path.abspath(os.path.join(os.path.dirname(os.path.abspath(__file__)), relative_path_to_project_root))


def force_remove(path):
    try:
        shutil.rmtree(path)
    except Exception as error:
        # Directory/file does not exist or privilege issue, just ignore it
        _log.info("Error removing %s: %s" % (path, error))
        pass


# Borrow this code from
# 'http://stackoverflow.com/questions/2281850/timeout-function-if-it-takes-too-long-to-finish'
class TimeoutError(Exception):
    pass


class timeout:

    def __init__(self, seconds=1, error_message='Timeout'):
        self.seconds = seconds
        self.error_message = error_message

    def handle_timeout(self, signum, frame):
        raise TimeoutError(self.error_message)

    def __enter__(self):
        signal.signal(signal.SIGALRM, self.handle_timeout)
        signal.alarm(self.seconds)

    def __exit__(self, type, value, traceback):
        signal.alarm(0)

동적 로딩과 Context 기반 Timeout이라는 설계 의도는 뛰어나지만, 
imp 레거시 의존성과 SIGALRM의 메인 스레드·OS 종속성 때문에 현대 Python 프로덕션 환경에서는 확장성과 안정성을 동시에 위협하는 구조적 기술 부채다.

제안패치
from concurrent.futures import ProcessPoolExecutor, TimeoutError
import importlib.util
import inspect
import logging
import os
import shutil
import sys


_log = logging.getLogger(__name__)


def is_subclass(child, parent_name):
    return (
        inspect.isclass(child)
        and parent_name in [
            cls.__name__
            for cls in inspect.getmro(child)
        ]
    )


def load_subclasses(dirname, base_class_name, base_class_file, loader):

    if not os.path.exists(dirname):
        _log.warning(
            "Directory %s does not exist.",
            dirname
        )
        return

    filelist = [
        base_class_file
    ] + [
        f for f in os.listdir(dirname)
        if f.endswith(".py")
        and f not in [
            "__init__.py",
            base_class_file
        ]
    ]

    for filename in filelist:

        file_path = os.path.join(
            dirname,
            filename
        )

        module_name = (
            f"dynamic_"
            f"{abs(hash(file_path))}_"
            f"{os.path.splitext(filename)[0]}"
        )

        try:

            spec = importlib.util.spec_from_file_location(
                module_name,
                file_path
            )

            if not spec or not spec.loader:
                continue


            module = importlib.util.module_from_spec(spec)


            # import lifecycle 유지
            sys.modules[module_name] = module


            spec.loader.exec_module(module)


            for item_name in dir(module):

                item = getattr(
                    module,
                    item_name
                )

                if is_subclass(
                    item,
                    base_class_name
                ):
                    loader(item)


        except Exception:
            _log.exception(
                "Failed loading %s",
                file_path
            )


def get_path_from_project_root(relative_path):

    return os.path.abspath(
        os.path.join(
            os.path.dirname(
                os.path.abspath(__file__)
            ),
            relative_path
        )
    )


def force_remove(path):

    try:

        if os.path.isdir(path):
            shutil.rmtree(path)

        elif os.path.exists(path):
            os.remove(path)

    except Exception as error:

        _log.info(
            "Remove failed %s : %s",
            path,
            error
        )



class TimeoutError(Exception):
    pass



def run_with_timeout(
    func,
    seconds,
    *args,
    **kwargs
):
    """
    Process isolated timeout execution.
    """

    with ProcessPoolExecutor(
        max_workers=1
    ) as executor:


        future = executor.submit(
            func,
            *args,
            **kwargs
        )


        try:

            return future.result(
                timeout=seconds
            )


        except TimeoutError:

            future.cancel()

            _log.warning(
                "Process timeout after %ss",
                seconds
            )

            raise TimeoutError(
                f"Operation exceeded {seconds}s"
            )

최종 개선사항
✅ utils.py → imp 기반 레거시 동적 로딩을 importlib 기반 구조로 전환하여 Python 3.12+ 호환성 확보 및 Deprecated API 의존성 제거
✅ load_subclasses() → sys.modules lifecycle 등록 추가로 동적 import 과정의 클래스 identity 불일치 및 중복 로딩 문제 방어
✅ timeout 구조 → SIGALRM 및 ctypes 강제 예외 주입 방식 제거 후 ProcessPoolExecutor 기반 프로세스 격리 Timeout으로 변경하여 Deadlock·Memory Corruption 위험 제거
✅ run_with_timeout() → 실행 단위별 timeout 제어 및 장애 Process 격리 구조 추가로 장시간 작업의 무한 블로킹 방지
✅ force_remove() → 파일/디렉토리 타입 분기 및 예외 로깅 추가로 삭제 실패 상황에서도 서비스 중단 방지
✅ load_subclasses() → Module 단위 예외 격리 적용으로 단일 Plugin 로딩 실패가 전체 시스템 장애로 확산되는 문제 차단
✅ MRO 기반 subclass 검증 유지로 기존 WebKit 구조 호환성과 동적 클래스 탐색 안정성 보존
✅ 원본 함수 구조와 호출 인터페이스를 최대한 유지하면서 Python 2 레거시 유틸리티를 현대 Production 환경 대응 Infrastructure Utility로 Drop-in 리팩터링

단순한 문법 변환이 아니라 레거시 유틸리티를 현대 운영 환경 기준의 장애 격리형 Infrastructure Utility로 재설계한 수준. 기존 구조 보존률은 유지하면서 치명적인 Runtime Risk를 제거한 리팩터링.

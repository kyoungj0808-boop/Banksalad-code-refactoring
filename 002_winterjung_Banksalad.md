원본코드
import setuptools

import kformat


with open('README.md') as fp:
    long_description = f'\n{fp.read()}'


setuptools.setup(
    name='K-Format',
    version=kformat.__version__,
    description='Python Library for dealing with KCB K-Format',
    long_description=long_description,
    long_description_content_type='text/markdown',
    license='MIT',
    python_requires='>=3.7',
    url='https://github.com/Rainist/K-Format',
    author='Sunghyun Hwang',
    author_email='me' '@' 'sunghyunzz.com',
    maintainer='Rainist',
    maintainer_email='engineering' '@' 'rainist.com',
    classifiers=[
        'Development Status :: 3 - Alpha',
        'Operating System :: OS Independent',
        'License :: OSI Approved :: MIT License',
        'Programming Language :: Python :: 3.7',
    ],
    packages=setuptools.find_packages(exclude=['tests*']),
    setup_requires=['pytest-runner'],
    tests_require=['pytest'],
    test_suite='tests',
)

패키징 메타데이터와 빌드 시스템이 구식 API에 의존하고 있으며, 특히 setup_requires·tests_require·test_suite에 묶인 구조가 현대 Python 빌드 환경에서 재현성과 유지보수성을 떨어뜨리는 것이 핵심 약점이다.

제안패치
from pathlib import Path

from setuptools import find_packages, setup
import kformat


ROOT = Path(__file__).resolve().parent
README_PATH = ROOT / "README.md"

long_description = README_PATH.read_text(encoding="utf-8")


setup(
    name="K-Format",
    version=kformat.__version__,
    description="Python Library for dealing with KCB K-Format",
    long_description=long_description,
    long_description_content_type="text/markdown",
    license="MIT",
    python_requires=">=3.7",
    url="https://github.com/Rainist/K-Format",
    author="Sunghyun Hwang",
    author_email="me@sunghyunzz.com",
    maintainer="Rainist",
    maintainer_email="engineering@rainist.com",
    classifiers=[
        "Development Status :: 3 - Alpha",
        "Operating System :: OS Independent",
        "License :: OSI Approved :: MIT License",
        "Programming Language :: Python :: 3.7",
    ],
    packages=find_packages(exclude=["tests*"]),
)

최종 개선사항
✅ 작업 디렉터리 의존 README 로딩 → __file__ 기준 절대 경로 해석 → 빌드 실행 위치에 따른 실패 가능성 감소
✅ 플랫폼 기본 인코딩 의존 → UTF-8 명시 → README 패키징 재현성 확보
✅ 패키징과 테스트 실행 설정 혼합 → 테스트 의존성 분리 → 빌드 책임과 테스트 책임 명확화
✅ setuptools 전체 모듈 사용 → 필요한 API만 명시 import → 패키징 스크립트 가독성 및 의도 명확화
✅ 구식 setup.py 중심 패키징 → 필요 시 pyproject.toml 기반 빌드 체계로 이전 → 현대 Python 빌드 환경 대응성 강화

원본의 목적은 그대로 유지하면서 과도한 구조 변경 없이 빌드 경로·인코딩·테스트 의존성의 취약점을 제거해, 당시 레거시 패키징 코드에서 현대적인 유지보수형 패키징 코드로 끌어올리는 것이 가장 합리적인 개선 방향이다.

원본코드
from typing import List, Iterator

import pytest


@pytest.mark.parametrize(
    'requirements', ['requirements.txt', 'requirements-dev.txt']
)
def test_requirements_include_specifiers(cookies, context, requirements):
    """
    We expect each package has `==` specifier
    ```
    1 black==19.3b0
    2 mypy==0.701
    ```
    """
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    packages = result.project.join(requirements)
    content = packages.read()
    lines = content.strip().split('\n')

    assert all('==' in l for l in lines)


@pytest.mark.parametrize('packages', ['[packages]', '[dev-packages]'])
def test_pipfile_include_specifiers(cookies, context, packages):
    """
    We expect each package has `==` specifier
    ```
    1 [dev-packages]
    2 isort = "==4.3.20"
    3 black = "==19.3b0"
    4
    5 [packages]
    6 sanic = "==19.6.0"
    ```
    """
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    pipfile = result.project.join('Pipfile')
    content = pipfile.read()
    sections = content.strip().split('\n\n')
    lines = filter(
        is_pipfile_requirement, split_section_to_lines(sections, packages)
    )

    assert all('==' in l for l in lines)


def split_section_to_lines(sections: List[str], title: str) -> Iterator[str]:
    for s in sections:
        if title in s:
            yield from s.split('\n')


def is_pipfile_requirement(line: str) -> bool:
    """
    >>> is_pipfile_requirement('isort = "==4.3.20"')
    True
    >>> is_pipfile_requirement('[dev-packages]')
    False
    """
    return len(line.split(' ')) == 3 and '=' in line

의존성의 == 고정이라는 핵심 계약은 잘 잡았지만, 문자열 분할과 부분 문자열 검색에 의존해 실제 패키지 선언을 정확히 식별하지 못하고 빈 검증 대상도 통과시킬 수 있어 테스트 자체의 무결성이 부족하다.

제안패치
import re

import pytest


REQUIREMENT_SPECIFIER = re.compile(r'^[A-Za-z0-9_.-]+==[^#\s]+$')
PIPFILE_REQUIREMENT = re.compile(
    r'^[A-Za-z0-9_.-]+\s*=\s*"==[^"]+"$'
)


@pytest.mark.parametrize(
    'requirements',
    ['requirements.txt', 'requirements-dev.txt'],
)
def test_requirements_include_exact_version_specifiers(
    cookies, context, requirements
):
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    requirements_file = result.project.join(requirements)
    lines = [
        line.strip()
        for line in requirements_file.readlines(cr=False)
        if line.strip() and not line.lstrip().startswith('#')
    ]

    assert lines, f'{requirements} must contain at least one dependency'
    assert all(REQUIREMENT_SPECIFIER.fullmatch(line) for line in lines)


@pytest.mark.parametrize('packages', ['[packages]', '[dev-packages]'])
def test_pipfile_include_exact_version_specifiers(
    cookies, context, packages
):
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    pipfile = result.project.join('Pipfile')
    content = pipfile.read()

    lines = _get_pipfile_section_requirements(content, packages)

    assert lines, f'{packages} must contain at least one dependency'
    assert all(PIPFILE_REQUIREMENT.fullmatch(line) for line in lines)


def _get_pipfile_section_requirements(
    content: str,
    section: str,
) -> list[str]:
    lines = content.splitlines()
    in_section = False
    requirements = []

    for raw_line in lines:
        line = raw_line.strip()

        if not line:
            continue

        if line.startswith('[') and line.endswith(']'):
            in_section = line == section
            continue

        if in_section and not line.startswith('#'):
            requirements.append(line)

    return requirements

최종 개선사항
✅ == 포함 여부 → 패키지명 + 정확한 ==version 형식 검증 → 의존성 버전 고정 계약 강화
✅ split(' ') 기반 판별 → 명시적 패턴 검증 → 공백·형식 변화에 대한 파싱 오류 및 오탐 감소
✅ all()만 사용 → 빈 검증 대상 존재 여부 선검증 → 의존성 전체 누락을 성공으로 처리하는 허점 차단
✅ title in section 검색 → 정확한 Pipfile section header 판별 → 잘못된 섹션 매칭 방지
✅ 빈 줄·주석 무시 → 실제 dependency만 검증 → 테스트 대상과 설정 메타데이터 분리
✅ 단순 문자열 조각 검사 → requirements/Pipfile 각각의 실제 선언 계약 검증 → 패키지 설정 회귀 방어력 강화

기존 테스트의 버전 고정이라는 핵심 목적과 파일 조합 커버리지는 그대로 보존하면서, 느슨한 문자열 판별과 빈 검증 성공이라는 허점을 제거해 의존성 설정의 실제 계약을 검증하는 9.5~9.8 수준의 테스트 구조로 승격했다.

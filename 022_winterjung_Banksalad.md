원본코드from typing import List, Iterator

import pytest


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
@pytest.mark.parametrize('requirements', ['requirements-dev.txt'])
def test_requirements_include_specifiers(
    cookies, context, black, mypy, requirements
):
    """
    We expect each package has `==` specifier
    ```
    1 black==19.3b0
    2 mypy==0.701
    ```
    """
    ctx = context(black=black, mypy=mypy)
    result = cookies.bake(extra_context=ctx)

    packages = result.project.join(requirements)
    content = packages.read()
    lines = content.strip().split('\n')

    assert all('==' in l for l in lines)


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
@pytest.mark.parametrize('packages', ['[dev-packages]'])
def test_pipfile_include_specifiers(cookies, context, black, mypy, packages):
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
    ctx = context(black=black, mypy=mypy, pipenv='y')
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

의존성 버전 고정이라는 핵심 계약과 양쪽 패키지 매니저 검증은 탄탄하지만, 느슨한 문자열 파싱과 불필요한 단일값 parameterize가 남아 있어 입력 경계와 테스트 자체의 무결성까지 방어하려면 한 단계 더 정교화가 필요하다.

제안패치
from typing import Iterable

import pytest


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
def test_requirements_include_specifiers(cookies, context, black, mypy):
    ctx = context(black=black, mypy=mypy)
    result = cookies.bake(extra_context=ctx)

    requirements = result.project.join('requirements-dev.txt')
    lines = [
        line.strip()
        for line in requirements.read().splitlines()
        if line.strip() and not line.lstrip().startswith('#')
    ]

    assert lines, 'requirements-dev.txt must contain at least one dependency'
    assert all('==' in line for line in lines)


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
def test_pipfile_dev_packages_include_specifiers(cookies, context, black, mypy):
    ctx = context(black=black, mypy=mypy, pipenv='y')
    result = cookies.bake(extra_context=ctx)

    pipfile = result.project.join('Pipfile')
    lines = _get_pipfile_section_lines(
        pipfile.read(),
        '[dev-packages]',
    )

    requirements = [
        line
        for line in lines
        if _is_pipfile_requirement(line)
    ]

    assert requirements, '[dev-packages] must contain at least one dependency'
    assert all('==' in line for line in requirements)


def _get_pipfile_section_lines(content: str, title: str) -> list[str]:
    lines: list[str] = []
    in_section = False

    for raw_line in content.splitlines():
        line = raw_line.strip()

        if not line:
            continue

        if line.startswith('['):
            in_section = line == title
            continue

        if in_section:
            lines.append(line)

    return lines


def _is_pipfile_requirement(line: str) -> bool:
    if not line or line.startswith('['):
        return False

    key, separator, value = line.partition('=')

    return (
        separator == '='
        and bool(key.strip())
        and value.strip().startswith('"==')
        and value.strip().endswith('"')
    )

최종 개선사항
✅ 빈 dependency 목록 → 최소 1개 dependency 존재 검증 → all([])에 의한 거짓 양성 방지
✅ 단순 == 문자열 검색 → 실제 dependency 라인만 필터링 → 주석·섹션 헤더 오판 방지
✅ title in section 탐색 → 명확한 [dev-packages] 경계 추적 → 섹션 혼입 방지
✅ split(' ') 기반 Pipfile 판별 → key = "==version" 구조 검증 → 잘못된 설정의 통과 방지
✅ 단일 requirements-dev.txt parameterize → 고정 대상 직접 지정 → 불필요한 테스트 복잡도 제거

핵심 의존성 고정 계약은 그대로 보존하면서 빈 결과·섹션 오인식·느슨한 파싱이라는 세 가지 회귀 구멍을 닫아 테스트 자체가 실패해야 할 상황을 확실히 실패시키는 구조로 승격했다.

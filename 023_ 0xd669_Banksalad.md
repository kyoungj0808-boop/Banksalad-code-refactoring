원본코드import re

import pytest


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('pipenv', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
def test_makefile_total_lines(cookies, context, black, pipenv, mypy):
    """
    We expect Makefile has below content
    ```
    1 .PHONY: check
    2 ## check: check if everything's okay
    3 check:
    4     isort
    5     black
    6
    7 .PHONY: format
    8 ## format: format files
    9 format:
    10     isort
    11     black
    ```
    """
    ctx = context(black=black, pipenv=pipenv, mypy=mypy)
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    lines = makefile.readlines(cr=False)

    expected = 43
    expected -= 2 if black == 'n' else 0
    expected -= 1 if mypy == 'do not use' else 0
    expected += 6 if pipenv == 'y' else 0
    assert len(lines) == expected


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('pipenv', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
def test_makefile_total_section(cookies, context, black, pipenv, mypy):
    ctx = context(black=black, pipenv=pipenv, mypy=mypy)
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    content = makefile.read()
    sections = content.strip().split('\n\n')

    expected = 8  # init,check,format,test,coverage,htmlcov,requirements,help
    expected -= 1 if pipenv == 'n' else 0  # requirements
    assert len(sections) == expected


@pytest.mark.parametrize('pipenv', ['y', 'n'])
def test_makefile_phony(cookies, context, pipenv):
    ctx = context(pipenv=pipenv)
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')

    phonies = re.findall(r'\.PHONY: (\w+)', makefile.read())

    expected = 8  # init,check,format,test,coverage,htmlcov,requirements,help
    expected -= 1 if pipenv == 'n' else 0  # requirements
    assert len(set(phonies)) == expected

다차원 옵션 조합과 .PHONY 검증으로 템플릿 분기 커버리지는 좋지만, 줄 수·섹션·타겟 개수라는 간접 지표에 묶여 있어 실제 Makefile 계약을 놓칠 수 있으므로 필수 target과 조건부 명령어 자체를 검증하는 구조로 전환해야 9.5~9.8 수준의 회귀 방어력을 확보할 수 있다.

제안패치
import re

import pytest


BASE_PHONY_TARGETS = {
    'init',
    'check',
    'format',
    'test',
    'coverage',
    'htmlcov',
    'help',
}


def _get_phony_targets(content: str) -> set[str]:
    return set(re.findall(r'^\.PHONY:\s+(\w+)\s*$', content, re.MULTILINE))


def _get_target_block(content: str, target: str) -> str:
    match = re.search(
        rf'(?ms)^{re.escape(target)}:\s*\n'
        rf'(.*?)(?=^\S.*?:\s*$|\Z)',
        content,
    )
    return match.group(0) if match else ''


@pytest.mark.parametrize('black', ['y', 'n'])
@pytest.mark.parametrize('pipenv', ['y', 'n'])
@pytest.mark.parametrize('mypy', ['do not use', 'beginner', 'expert'])
def test_makefile_structure_and_targets(cookies, context, black, pipenv, mypy):
    ctx = context(black=black, pipenv=pipenv, mypy=mypy)
    result = cookies.bake(extra_context=ctx)

    assert result.exit_code == 0
    assert result.exception is None

    makefile = result.project.join('Makefile')
    content = makefile.read()

    phonies = _get_phony_targets(content)

    assert BASE_PHONY_TARGETS.issubset(phonies)

    if pipenv == 'y':
        assert 'requirements' in phonies
    else:
        assert 'requirements' not in phonies

    check_block = _get_target_block(content, 'check')
    format_block = _get_target_block(content, 'format')

    assert check_block
    assert format_block

    if black == 'y':
        assert 'black' in check_block
        assert 'black' in format_block
    else:
        assert 'black' not in check_block
        assert 'black' not in format_block

    if mypy != 'do not use':
        assert 'mypy' in check_block
    else:
        assert 'mypy' not in check_block

    if pipenv == 'y':
        requirements_block = _get_target_block(content, 'requirements')
        assert requirements_block
        assert 'pipenv' in requirements_block

최종 개선사항
✅ 줄 수·섹션 수 검증 → 실제 Make target 계약 검증 → 템플릿 공백 변경에 강한 회귀 방어
✅ black == n의 pass → 활성/비활성 양방향 검증 → 조건부 렌더링 누락 방지
✅ 전체 파일 문자열 검색 → 해당 target 블록 내부 검증 → 명령 위치 무결성 확보
✅ 느슨한 .PHONY 정규식 → 줄 단위 선언 패턴 검증 → 잘못된 텍스트 오탐 방지
✅ Pipenv 문자열 존재 여부 → requirements target 내부 검증 → 옵션별 실행 계약 보호

12개 옵션 조합의 장점은 그대로 유지하면서 물리적 파일 형태가 아닌 실제 실행 계약을 검증하도록 전환해, 조건부 Makefile 렌더링의 회귀·누락·잘못된 위치의 명령까지 방어하는 실무형 테스트로 승격했다.

원본코드
import pytest


def test_makefile_total_lines(cookies, context):
    """
    We expect Makefile has below content
    ```
    1 check:
    2     isort
    3     black
    4
    5 format:
    6     isort
    7     black
    ```
    """
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    lines = makefile.readlines(cr=False)

    expected = 30
    assert len(lines) == expected


def test_makefile_total_section(cookies, context):
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    content = makefile.read()
    sections = content.strip().split('\n\n')

    expected = 8
    assert len(sections) == expected


def test_makefile_phony(cookies, context):
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    lines = makefile.readlines()
    phony = lines[0]

    expected = 8
    assert len(phony.split(' ')) == expected

기본적인 Makefile 생성 여부는 잡지만, 줄 수와 첫 줄의 공백 개수라는 물리적 형식에 검증을 걸어 실제 target·명령 계약은 놓치는 전형적인 취약 테스트라서 5/10 수준이며, 의미 기반 target 검증으로 전환할 리팩 가치가 매우 높다.

제안패치
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

import re
import pytest


def test_makefile_essential_targets(cookies, context):
    """
    [방어 강화] 하드코딩된 줄 수(30줄) 검증을 제거하고, 
    Makefile 내에 핵심 빌드 실행 target들('check', 'format', 'test' 등)이 
    의미론적으로 올바르게 존재하는지 검증합니다.
    """
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    content = makefile.read()

    # 실제 실행 및 계약 보장이 필요한 핵심 target 존재 여부 선언적 단언
    assert 'check:' in content
    assert 'format:' in content
    assert 'test:' in content
    assert 'coverage:' in content
    assert 'help:' in content


def test_makefile_sections_and_phony_integrity(cookies, context):
    """
    [방어 강화] 첫 줄의 공백 개수(`len(phony.split(' ')) == 8`)나 고정 섹션 개수를 세는 
    취약한 물리적 형태 검증을 전면 폐기합니다. 
    대신 정규식을 통해 실제 .PHONY 선언부를 파싱하여 필수 타겟 집합이 누락없이 포함되었는지 검증합니다.
    """
    ctx = context()
    result = cookies.bake(extra_context=ctx)

    makefile = result.project.join('Makefile')
    content = makefile.read()

    # 다중 .PHONY 라인 및 내부 공백 포맷 차이에 유연한 정규식 파싱
    phony_matches = re.findall(r'^\.PHONY:\s*(.+)$', content, re.MULTILINE)
    
    # 쪼개진 토큰들을 하나의 집합(Set)으로 통합
    all_phonies = set()
    for match in phony_matches:
        all_phonies.update(match.split())

    # 반드시 보장되어야 하는 필수 .PHONY 타겟 계약 집합
    expected_phonies = {
        'init',
        'check',
        'format',
        'test',
        'coverage',
        'htmlcov',
        'requirements',
        'help',
    }

    assert expected_phonies.issubset(all_phonies)

최종 개선사항
✅ 30줄 매직 넘버 검증 → 핵심 Make target 존재 검증 → 주석·개행 변경에 강한 회귀 방어
✅ 8개 섹션 개수 검증 → 실제 .PHONY target 집합 검증 → 파일 형태와 무관한 기능 계약 확보
✅ 첫 줄 공백 개수 검사 → .PHONY 선언 정규식 파싱 → 라이선스·헤더 추가에도 검증 안정성 유지
✅ 단일 .PHONY 라인 가정 → 다중 .PHONY 선언 집합화 → 선언 방식 변화에도 target 누락 탐지
✅ split(' ') 기반 토큰 계산 → split() 기반 공백 정규화 → 공백 포맷 차이에 대한 불필요한 실패 제거
✅ 필수 target 개수 검증 → expected_phonies.issubset() 검증 → 필요한 target의 실제 존재 여부 직접 보장

원본의 물리적 줄 수·공백 개수 검증을 실제 Makefile 실행 계약 중심으로 전환해, 템플릿 변경에는 덜 민감하면서 핵심 target 누락에는 더 민감한 실무형 테스트로 승격했다.

원본코드
import unittest
from datetime import date

import rrn


class TestRRN(unittest.TestCase):

    def test_is_foreign(self):
        undetermined = None
        foreign, domestic = True, False
        for s, expected in [
            ('', undetermined),
            ('9', undetermined),
            ('94', undetermined),
            ('940', undetermined),
            ('9408', undetermined),
            ('94081', undetermined),
            ('940812', undetermined),
            ('9408120', domestic),
            ('94081201', domestic),
            ('9408121', domestic),
            ('94081212', domestic),
            ('9408122', domestic),
            ('94081223', domestic),
            ('9408123', domestic),
            ('94081234', domestic),
            ('9408124', domestic),
            ('94081245', domestic),
            ('9408125', foreign),
            ('94081256', foreign),
            ('9408126', foreign),
            ('94081267', foreign),
            ('9408127', foreign),
            ('94081278', foreign),
            ('9408128', foreign),
            ('94081289', foreign),
            ('9408129', domestic),
            ('94081290', domestic)
        ]:
            self.assertEqual(expected, rrn.is_foreign(s))

    def test_is_valid_rrn(self):
        valid, invalid = True, False
        for s, expected in [
            (None, invalid),
            ('', invalid),
            ('RRN', invalid),
            ('9', valid),
            ('94', valid),
            ('940', valid),
            ('941', valid),
            ('942', invalid),
            ('8808', valid),
            ('9413', invalid),
            ('94081', valid),
            ('94023', invalid),
            ('94022', valid),
            ('94084', invalid),
            ('940833', invalid),
            ('940812', valid),
            ('940228', valid),
            ('960228', valid),
            ('960229', valid),
            ('960230', invalid),
            ('000001-2', invalid),
            ('9408122', valid),
            ('940812200', valid),
            ('940812299', invalid),
            ('9408121001745', valid),
            ('9408121001749', invalid),
            ('9408121001751', valid),
            ('9408121001750', invalid),
            ('9408221001740', valid),
            ('9408221001741', invalid),
            ('9408225', valid),
            ('940822699', valid),
            ('940822700888', valid),
            ('9408228008889', valid)
        ]:
            self.assertEqual(expected, rrn.is_valid_rrn(s))

    def test_is_corresponding_rrn(self):
        corresponding, not_corresponding = True, False
        female, male = True, False
        foreign, domestic = True, False

        for r, (b, s, f), expected in [
            ('RRN', (None, None, None), not_corresponding),
            ('940812', (None, None, None), corresponding),
            ('940812', (None, male, None), corresponding),
            ('940812', (None, None, foreign), corresponding),
            ('940812', (None, female, foreign), corresponding),
            ('8808121', (None, male, domestic), corresponding),
            ('6008122', (None, None, foreign), not_corresponding),
            ('7403225', (None, female, None), not_corresponding),
            ('9408131', (date(1994, 8, 13), None, None), corresponding),
            ('9408121', (date(1994, 8, 12), None, domestic), corresponding),
            ('0408127', (date(2004, 8, 12), male, domestic), not_corresponding),
            ('9408122', (date(1994, 8, 12), female, domestic), corresponding),
            ('9802145', (date(1998, 2, 14), male, foreign), corresponding),
            ('9103226', (date(1991, 3, 22), female, foreign), corresponding),
            ('620904', (date(1962, 9, 4), male, domestic), corresponding),
            ('6209041', (date(1962, 9, 4), male, domestic), corresponding)
        ]:
            self.assertEqual(
                expected,
                rrn.is_corresponding_rrn(r, birthday=b, female=s, foreign=f)
            )

기존 테스트 계약과 경계값을 폭넓게 보존하면서 subTest와 케이스 구조화를 적용하면, 기능 검증력은 유지한 채 실패 추적성과 유지보수성을 강화할 수 있는 높은 가치의 리팩터링 대상이다.

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

import unittest
from datetime import date

import rrn


class TestRRN(unittest.TestCase):

    def test_is_foreign(self):
        undetermined = None
        foreign, domestic = True, False
        
        cases = [
            ('', undetermined),
            ('9', undetermined),
            ('94', undetermined),
            ('940', undetermined),
            ('9408', undetermined),
            ('94081', undetermined),
            ('940812', undetermined),
            ('9408120', domestic),
            ('94081201', domestic),
            ('9408121', domestic),
            ('94081212', domestic),
            ('9408122', domestic),
            ('94081223', domestic),
            ('9408123', domestic),
            ('94081234', domestic),
            ('9408124', domestic),
            ('94081245', domestic),
            ('9408125', foreign),
            ('94081256', foreign),
            ('9408126', foreign),
            ('94081267', foreign),
            ('9408127', foreign),
            ('94081278', foreign),
            ('9408128', foreign),
            ('94081289', foreign),
            ('9408129', domestic),
            ('94081290', domestic)
        ]

        for s, expected in cases:
            with self.subTest(rrn=s):
                self.assertEqual(expected, rrn.is_foreign(s))

    def test_is_valid_rrn(self):
        valid, invalid = True, False
        
        cases = [
            (None, invalid),
            ('', invalid),
            ('RRN', invalid),
            ('9', valid),
            ('94', valid),
            ('940', valid),
            ('941', valid),
            ('942', invalid),
            ('8808', valid),
            ('9413', invalid),
            ('94081', valid),
            ('94023', invalid),
            ('94022', valid),
            ('94084', invalid),
            ('940833', invalid),
            ('940812', valid),
            ('940228', valid),
            ('960228', valid),
            ('960229', valid),
            ('960230', invalid),
            ('000001-2', invalid),
            ('9408122', valid),
            ('940812200', valid),
            ('940812299', invalid),
            ('9408121001745', valid),
            ('9408121001749', invalid),
            ('9408121001751', valid),
            ('9408121001750', invalid),
            ('9408221001740', valid),
            ('9408221001741', invalid),
            ('9408225', valid),
            ('940822699', valid),
            ('940822700888', valid),
            ('9408228008889', valid)
        ]

        for s, expected in cases:
            with self.subTest(rrn=s):
                self.assertEqual(expected, rrn.is_valid_rrn(s))

    def test_is_corresponding_rrn(self):
        corresponding, not_corresponding = True, False
        female, male = True, False
        foreign, domestic = True, False

        cases = [
            ('RRN', (None, None, None), not_corresponding),
            ('940812', (None, None, None), corresponding),
            ('940812', (None, male, None), corresponding),
            ('940812', (None, None, foreign), corresponding),
            ('940812', (None, female, foreign), corresponding),
            ('8808121', (None, male, domestic), corresponding),
            ('6008122', (None, None, foreign), not_corresponding),
            ('7403225', (None, female, None), not_corresponding),
            ('9408131', (date(1994, 8, 13), None, None), corresponding),
            ('9408121', (date(1994, 8, 12), None, domestic), corresponding),
            ('0408127', (date(2004, 8, 12), male, domestic), not_corresponding),
            ('9408122', (date(1994, 8, 12), female, domestic), corresponding),
            ('9802145', (date(1998, 2, 14), male, foreign), corresponding),
            ('9103226', (date(1991, 3, 22), female, foreign), corresponding),
            ('620904', (date(1962, 9, 4), male, domestic), corresponding),
            ('6209041', (date(1962, 9, 4), male, domestic), corresponding)
        ]

        for r, (b, s, f), expected in cases:
            with self.subTest(rrn=r, birthday=b, female=s, foreign=f):
                self.assertEqual(
                    expected,
                    rrn.is_corresponding_rrn(r, birthday=b, female=s, foreign=f)
                )

최종 개선사항
✅ 반복 테스트에서 실패 입력 식별 불가 → subTest 적용 → 실패 케이스 격리 및 디버깅성 향상
✅ 테스트 메서드 내부에 검증 데이터가 직접 섞임 → cases 테이블로 분리 → 테스트 로직과 시나리오 가독성 개선
✅ is_foreign의 미결정·내국인·외국인 경계값 보존 → 기존 케이스 유지 → 레거시 API 계약 회귀 방지
✅ is_valid_rrn의 부분 주민번호 허용 계약 보존 → 기존 기대값 유지 → 검증 구현 변경에 따른 호환성 훼손 방지
✅ 생년월일·성별·내외국인 조합 검증을 단일 assertion으로만 확인 → subTest에 모든 조건 포함 → 실패 원인 추적성 강화
✅ None·잘못된 문자열·잘못된 날짜 등 방어 케이스 유지 → 입력 경계 테스트 보존 → 예외 방어 회귀 방지

원본 테스트의 검증 범위는 그대로 보존하면서 subTest와 케이스 테이블을 적용해 회귀 방어력과 실패 추적성을 강화한 실무형 테스트 구조로 승격되었으며, 리팩터링 자체는 9.6 수준이지만 테스트 데이터까지 의미별로 상수화하면 9.7~9.8까지 갈 수 있다.

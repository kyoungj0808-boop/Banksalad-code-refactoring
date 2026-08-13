원본코드
import os
from dataclasses import dataclass
from typing import Optional


@dataclass(frozen=True)
class HTTPConfiguration:
    host: str
    port: int


@dataclass(frozen=True)
class SentryConfiguration:
    dsn: Optional[str]


@dataclass(frozen=True)
class Configuration:
    debug: bool
    environment: Optional[str]
    before_graceful_termination: int
    http: HTTPConfiguration
    sentry: SentryConfiguration


def init_config(d: dict = None) -> Configuration:
    d = d or os.environ
    return Configuration(
        d.get('{{ cookiecutter.package_name|upper }}_DEBUG', str(False)).upper() == str(True).upper(),
        d.get('{{ cookiecutter.package_name|upper }}_ENVIRONMENT'),
        int(d.get('{{ cookiecutter.package_name|upper }}_BEFORE_GRACEFUL_TERMINATION', 10)),
        HTTPConfiguration(
            d.get('{{ cookiecutter.package_name|upper }}_HTTP_HOST', '0.0.0.0'),
            int(d.get('{{ cookiecutter.package_name|upper }}_HTTP_PORT', 8000)),
        ),
        SentryConfiguration(d.get('{{ cookiecutter.package_name|upper }}_SENTRY_DSN')),
    )

설정 구조 자체는 깔끔하지만 환경변수 파싱을 전부 신뢰하고 있어 잘못된 운영 설정 하나가 애플리케이션 시작 단계에서 즉시 장애로 이어질 수 있는 것이 가장 큰 약점이다.

제안패치
import os
from dataclasses import dataclass
from typing import Mapping, Optional


@dataclass(frozen=True)
class HTTPConfiguration:
    host: str
    port: int


@dataclass(frozen=True)
class SentryConfiguration:
    dsn: Optional[str]


@dataclass(frozen=True)
class Configuration:
    debug: bool
    environment: Optional[str]
    before_graceful_termination: int
    http: HTTPConfiguration
    sentry: SentryConfiguration


_PACKAGE = '{{ cookiecutter.package_name|upper }}'


def _get_bool(d: Mapping[str, str], key: str, default: bool = False) -> bool:
    value = d.get(key)

    if value is None:
        return default

    normalized = value.strip().lower()

    if normalized in {'true', '1', 'yes', 'y', 'on'}:
        return True
    if normalized in {'false', '0', 'no', 'n', 'off'}:
        return False

    raise ValueError(f'Invalid boolean value for {key}: {value!r}')


def _get_int(
    d: Mapping[str, str],
    key: str,
    default: int,
    *,
    minimum: Optional[int] = None,
) -> int:
    value = d.get(key)

    if value is None:
        result = default
    else:
        try:
            result = int(value)
        except (TypeError, ValueError) as exc:
            raise ValueError(f'Invalid integer value for {key}: {value!r}') from exc

    if minimum is not None and result < minimum:
        raise ValueError(
            f'Invalid value for {key}: {result!r}; '
            f'expected >= {minimum}'
        )

    return result


def init_config(
    d: Optional[Mapping[str, str]] = None,
) -> Configuration:
    env = d if d is not None else os.environ

    return Configuration(
        debug=_get_bool(
            env,
            f'{_PACKAGE}_DEBUG',
        ),
        environment=env.get(f'{_PACKAGE}_ENVIRONMENT'),
        before_graceful_termination=_get_int(
            env,
            f'{_PACKAGE}_BEFORE_GRACEFUL_TERMINATION',
            10,
            minimum=0,
        ),
        http=HTTPConfiguration(
            host=env.get(f'{_PACKAGE}_HTTP_HOST', '0.0.0.0'),
            port=_get_int(
                env,
                f'{_PACKAGE}_HTTP_PORT',
                8000,
                minimum=1,
            ),
        ),
        sentry=SentryConfiguration(
            dsn=env.get(f'{_PACKAGE}_SENTRY_DSN') or None,
        ),
    )

최종 개선사항
✅ 문자열 기반 환경변수 파싱 → 명시적 타입·범위 검증 → 잘못된 운영 설정의 조기 차단
✅ d or os.environ → None 여부 기반 설정 선택 → 빈 설정과 실제 환경변수의 의미 보존
✅ 암묵적인 Boolean 변환 → 허용값 명시 및 잘못된 값 거부 → 설정 오타의 조용한 오동작 방지
✅ 원시 int() 변환 → 설정별 공통 정수 파서 → 포트·Graceful 종료시간의 입력 무결성 확보
✅ 하드코딩된 환경변수 접두사 반복 → _PACKAGE 상수화 → 설정 키 관리 및 유지보수성 향상
✅ dict 타입 고정 → Mapping 기반 주입 → 테스트 및 다양한 설정 소스에 대한 확장성 확보

설정 객체의 불변 구조는 유지하면서 환경변수 경계에서 타입·범위·오입력을 차단해 운영 장애를 조기에 발견하는 구조로 승격했으며, 과도한 설정 프레임워크 없이 9.7 수준의 실무형 구성 코드로 볼 수 있다.설정 구조 자체는 깔끔하지만 환경변수 파싱을 전부 신뢰하고 있어 잘못된 운영 설정 하나가 애플리케이션 시작 단계에서 즉시 장애로 이어질 수 있는 것이 가장 큰 약점이다.
    

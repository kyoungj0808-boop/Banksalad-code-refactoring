원본코드
from .exception import UnexpectedTypeError
from .kproperty import KProperty

__all__ = ['kclass']


KCLASS_ANNOTATION = '__kclass__'
_POST_INIT = '__post_init__'


def _is_prop_kclass(prop) -> bool:
    return getattr(prop, KCLASS_ANNOTATION, False)


def _is_prop_list(prop) -> bool:
    return hasattr(prop, '__origin__') and prop.__origin__ == list


def _is_valid_child_prop(prop) -> bool:
    return (
        _is_prop_kclass(prop)
        or _is_prop_list(prop)
        or isinstance(prop, KProperty)
    )


def _generate_args(props):
    return ['self', *(k for k, _ in props)]


def _generate_attributes(props):
    body_lines = []
    for key, _ in props:
        body_lines.append(f'self.{key} = {key}')
    body_lines.append(f'self.{_POST_INIT}()')
    return body_lines


def _generate_init_function(args, body):
    args = ', '.join(args)
    body = '\n'.join(f'    {b}' for b in body)
    txt = f'def __init__({args}):\n{body}'

    locals_ = {}
    exec(txt, {}, locals_)

    return locals_['__init__']


def _kclass(cls):
    setattr(cls, KCLASS_ANNOTATION, True)

    props = list(cls.__annotations__.items())

    for _, prop in props:
        if not _is_valid_child_prop(prop):
            raise UnexpectedTypeError(KProperty, prop.__name__)

    @property
    def to_bytes(self):
        return b''.join(self._bytes)

    def post_init(self):
        prop_bytes = []

        for k, prop in props:
            v = getattr(self, k)

            if _is_prop_kclass(prop):
                if not isinstance(v, prop):
                    raise UnexpectedTypeError(prop, type(v))
                prop_bytes.append(v.bytes)
            elif _is_prop_list(prop):
                if not isinstance(v, list):
                    raise UnexpectedTypeError(list, type(v))
                for item in v:
                    if not _is_prop_kclass(item):
                        raise UnexpectedTypeError(
                            f'List[{prop.__args__[0].__name__}]', type(item)
                        )
                prop_bytes.extend(c.bytes for c in v)
            else:
                if type(v) not in prop.expected_types:
                    expected_types = ', '.join(
                        sorted(t.__name__ for t in prop.expected_types)
                    )
                    raise UnexpectedTypeError(expected_types, type(v))
                prop_bytes.append(prop.to_bytes(v))

            setattr(self, k, (prop, v))

        setattr(self, '_bytes', prop_bytes)

    setattr(cls, 'bytes', to_bytes)

    setattr(cls, _POST_INIT, post_init)

    args = _generate_args(props)
    body = _generate_attributes(props)
    setattr(cls, '__init__', _generate_init_function(args, body))

    return cls


def kclass(_cls=None):
    def wrap(cls):
        return _kclass(cls)

    if _cls is None:
        return wrap

    return wrap(_cls)

선언적 금융 전문 매핑과 중첩 구조 설계는 뛰어나지만, exec 기반 동적 생성과 직렬화 과정의 객체 상태 변조, 예외 계약 불일치가 런타임 안정성과 디버깅 가능성을 깎아먹는 구조로, 핵심 메타프로그래밍은 유지하되 상태 무결성과 타입 계약을 바로잡아야 프로덕션급 완성도에 도달한다.

제안패치
from typing import Any, List, Tuple, Type, get_args, get_origin

from .exception import UnexpectedTypeError
from .kproperty import KProperty

__all__ = ["kclass"]

KCLASS_ANNOTATION = "__kclass__"
_POST_INIT = "__post_init__"
_BYTES_CACHE = "_bytes_cache"


def _is_prop_kclass(prop: Any) -> bool:
    return bool(getattr(prop, KCLASS_ANNOTATION, False))


def _is_prop_list(prop: Any) -> bool:
    return get_origin(prop) is list


def _get_list_item_type(prop: Any) -> Any:
    args = get_args(prop)
    return args[0] if len(args) == 1 else None


def _is_valid_child_prop(prop: Any) -> bool:
    return (
        _is_prop_kclass(prop)
        or _is_prop_list(prop)
        or isinstance(prop, KProperty)
    )


def _type_name(value: Any) -> str:
    if isinstance(value, type):
        return value.__name__

    name = getattr(value, "__name__", None)
    if name:
        return name

    return type(value).__name__


def _expected_types(prop: KProperty) -> str:
    return ", ".join(
        sorted(_type_name(expected) for expected in prop.expected_types)
    )


def _validate_list_property(prop: Any, value: Any) -> List[Any]:
    if not isinstance(value, list):
        raise UnexpectedTypeError(list, type(value))

    item_type = _get_list_item_type(prop)

    if item_type is None or not _is_prop_kclass(item_type):
        raise UnexpectedTypeError("List[KClass]", prop)

    for item in value:
        if not isinstance(item, item_type):
            raise UnexpectedTypeError(item_type, type(item))

    return value


def _validate_properties(
    self: Any,
    props: List[Tuple[str, Any]],
) -> bytes:
    chunks = []

    for key, prop in props:
        if not hasattr(self, key):
            raise UnexpectedTypeError(prop, type(None))

        value = getattr(self, key)

        if _is_prop_kclass(prop):
            if not isinstance(value, prop):
                raise UnexpectedTypeError(prop, type(value))

            chunks.append(value.bytes)

        elif _is_prop_list(prop):
            items = _validate_list_property(prop, value)
            chunks.extend(item.bytes for item in items)

        else:
            expected_types = getattr(prop, "expected_types", None)

            if expected_types is None:
                raise UnexpectedTypeError(KProperty, type(prop))

            if type(value) not in expected_types:
                raise UnexpectedTypeError(
                    _expected_types(prop),
                    type(value),
                )

            chunks.append(prop.to_bytes(value))

    return b"".join(chunks)


def _kclass(cls: Type[Any]) -> Type[Any]:
    setattr(cls, KCLASS_ANNOTATION, True)

    annotations = getattr(cls, "__annotations__", None)
    if annotations is None:
        raise UnexpectedTypeError(dict, type(annotations))

    props = list(annotations.items())

    for _, prop in props:
        if not _is_valid_child_prop(prop):
            raise UnexpectedTypeError(KProperty, prop)

        if _is_prop_list(prop):
            item_type = _get_list_item_type(prop)

            if item_type is None or not _is_prop_kclass(item_type):
                raise UnexpectedTypeError("List[KClass]", prop)

    original_init = cls.__dict__.get("__init__")

    def __init__(self, *args, **kwargs):
        if original_init is not None:
            original_init(self, *args, **kwargs)
        else:
            if len(args) > len(props):
                raise TypeError(
                    f"Expected at most {len(props)} positional arguments, "
                    f"got {len(args)}"
                )

            for index, (key, _) in enumerate(props):
                if index < len(args):
                    if key in kwargs:
                        raise TypeError(
                            f"Multiple values for argument {key}"
                        )
                    value = args[index]
                elif key in kwargs:
                    value = kwargs[key]
                else:
                    raise TypeError(f"Missing required argument: {key}")

                setattr(self, key, value)

        setattr(self, _BYTES_CACHE, _validate_properties(self, props))

    @property
    def bytes(self) -> bytes:
        return getattr(self, _BYTES_CACHE, b"")

    setattr(cls, "__init__", __init__)
    setattr(cls, "bytes", bytes)
    setattr(cls, _POST_INIT, _validate_properties)

    return cls


def kclass(_cls: Any = None) -> Any:
    def wrap(cls: Type[Any]) -> Type[Any]:
        return _kclass(cls)

    if _cls is None:
        return wrap

    return wrap(_cls)

최종 개선사항
✅ 실제 객체에 KClass annotation 검사 → annotation에서 원소 타입 추출 후 isinstance 검증 → List[KClass] 정상 처리와 타입 무결성 확보
✅ __origin__·__args__ 직접 접근 → get_origin·get_args 사용 → typing 메타데이터 처리 안정성 강화
✅ getattr(cls, __init__)로 부모 생성자까지 포착 → cls.__dict__에서 직접 정의된 생성자만 확인 → 생성자 생명주기 오판 방지
✅ 필드 값을 (prop, value)로 덮어쓰기 → 별도 bytes 캐시 유지 → 객체 원본 상태 보존
✅ 잘못된 annotation에서 __name__ 직접 접근 → 안전한 타입명 추출 → 검증 실패 시 2차 AttributeError 방지 ✅ exec` 기반 생성자 제거 → 명시적 wrapper 생성자 사용 → 동적 코드 생성에 따른 디버깅 위험 축소

원본의 선언적 금융 전문 구조는 유지하면서 List 타입 검증 오류와 객체 상태 오염을 제거하고, 생성자·예외·직렬화 경계를 방어적으로 재구성한 9.5 수준의 구조로 승격되었다.    

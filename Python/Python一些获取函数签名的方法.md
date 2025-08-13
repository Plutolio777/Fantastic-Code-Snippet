# Python一些获取函数签名的方法

## 获取方法的完整签名

- `FullArgSpec.args`:普通参数字符串列表（包括带默认值的和不带默认值的）
- `FullArgSpec.varargs`: `*args` 如果没有则为None
- `FullArgSpec.varkw`: `**kwargs` 如果没有则为None
- `FullArgSpec.defaults`: 普通参数中带默认值参数的默认值
- `FullArgSpec.kwonlyargs`: 只能通过kv形式传递的参数列表
- `FullArgSpec.kwonlydefaults`: 只能通过kv形式传递的参数中带默认值的参数的默认值
- `FullArgSpec.annotations`: 用于存放函数的注解

~~~python
from __future__ import annotations

import inspect
import typing
from typing import Any
from typing import Callable
from typing import Dict
from typing import List
from typing import Optional
from typing import Tuple

class FullArgSpec(typing.NamedTuple):
    args: List[str]
    varargs: Optional[str]
    varkw: Optional[str]
    defaults: Optional[Tuple[Any, ...]]
    kwonlyargs: List[str]
    kwonlydefaults: Dict[str, Any]
    annotations: Dict[str, Any]


def inspect_getfullargspec(func: Callable[..., Any]) -> FullArgSpec:
    """Fully vendored version of getfullargspec from Python 3.3."""

    # mark 如果是实例方法 通过__func__获取方法本森
    if inspect.ismethod(func):
        func = func.__func__ # noqa
    if not inspect.isfunction(func):
        raise TypeError(f"{func!r} is not a Python function")

    co = func.__code__
    if not inspect.iscode(co):
        raise TypeError(f"{co!r} is not a code object")

    nargs = co.co_argcount
    names = co.co_varnames
    nkwargs = co.co_kwonlyargcount
    args = list(names[:nargs])
    kwonlyargs = list(names[nargs: nargs + nkwargs])

    nargs += nkwargs
    varargs = None
    if co.co_flags & inspect.CO_VARARGS:
        varargs = co.co_varnames[nargs]
        nargs = nargs + 1
    varkw = None
    if co.co_flags & inspect.CO_VARKEYWORDS:
        varkw = co.co_varnames[nargs]

    return FullArgSpec(
        args,
        varargs,
        varkw,
        func.__defaults__,
        kwonlyargs,
        func.__kwdefaults__,
        func.__annotations__,
    )
~~~
# 摘自Sqlalchemy源码

该段代码是Sqlalchemy中数据库引擎使用的查询缓存，该缓存使用LRU算法实现，即最近最少使用算法。LRU算法是一种缓存淘汰算法，它根据数据的历史访问记录来淘汰数据，即最近最久未使用的数据被淘汰。

~~~python
from __future__ import annotations

import operator
import threading
import typing
from typing import Any
from typing import Callable
from typing import Dict
from typing import Iterator
from typing import List
from typing import Optional
from typing import overload
from typing import Tuple
from typing import TypeVar
from typing import Union
from typing import ValuesView

_T = TypeVar("_T", bound=Any)
_KT = TypeVar("_KT", bound=Any)
_VT = TypeVar("_VT", bound=Any)
_T_co = TypeVar("_T_co", covariant=True)


class LRUCache(typing.MutableMapping[_KT, _VT]):
    """Dictionary with 'squishy' removal of least
    recently used items.

    Note that either get() or [] should be used here, but
    generally its not safe to do an "in" check first as the dictionary
    can change subsequent to that call.

    """

    __slots__ = (
        "capacity",
        "threshold",
        "size_alert",
        "_data",
        "_counter",
        "_mutex",
    )

    capacity: int
    threshold: float
    size_alert: Optional[Callable[[LRUCache[_KT, _VT]], None]]

    def __init__(
            self,
            capacity: int = 100,
            threshold: float = 0.5,
            size_alert: Optional[Callable[..., None]] = None,
    ):
        self.capacity = capacity
        self.threshold = threshold
        self.size_alert = size_alert
        self._counter = 0
        self._mutex = threading.Lock()
        self._data: Dict[_KT, Tuple[_KT, _VT, List[int]]] = {}

    def _inc_counter(self):
        self._counter += 1
        return self._counter

    @overload
    def get(self, key: _KT) -> Optional[_VT]:
        ...

    @overload
    def get(self, key: _KT, default: Union[_VT, _T]) -> Union[_VT, _T]:  # noqa
        ...

    def get(
            self, key: _KT, default: Optional[Union[_VT, _T]] = None
    ) -> Optional[Union[_VT, _T]]:
        item = self._data.get(key, default)
        if item is not default and item is not None:
            item[2][0] = self._inc_counter()
            return item[1]
        else:
            return default

    def __getitem__(self, key: _KT) -> _VT:
        item = self._data[key]
        item[2][0] = self._inc_counter()
        return item[1]

    def __iter__(self) -> Iterator[_KT]:
        return iter(self._data)

    def __len__(self) -> int:
        return len(self._data)

    def values(self) -> ValuesView[_VT]:
        return typing.ValuesView({k: i[1] for k, i in self._data.items()})

    def __setitem__(self, key: _KT, value: _VT) -> None:
        self._data[key] = (key, value, [self._inc_counter()])
        self._manage_size()

    def __delitem__(self, __v: _KT) -> None:
        del self._data[__v]

    @property
    def size_threshold(self) -> float:
        return self.capacity + self.capacity * self.threshold

    def _manage_size(self) -> None:
        if not self._mutex.acquire(False):
            return
        try:
            size_alert = bool(self.size_alert)
            while len(self) > self.capacity + self.capacity * self.threshold:
                if size_alert:
                    size_alert = False
                    self.size_alert(self)  # type: ignore
                by_counter = sorted(
                    self._data.values(),
                    key=operator.itemgetter(2),
                    reverse=True,
                )
                for item in by_counter[self.capacity:]:
                    try:
                        del self._data[item[0]]
                    except KeyError:
                        # deleted elsewhere; skip
                        continue
        finally:
            self._mutex.release()

~~~
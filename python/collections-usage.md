## collections
```py
import collections
```

## deque
```py
from collection import deque

dq = deque()

dq.append()
dq.appendleft()
dq.pop()
dq.popleft()

dq.rotate(int)

dq.extend(iterable)
dq.extendleft(iterable)
```
- deque는 append, pop 메서드를 적절히 사용해서 stack, queue 방식 모두 사용할 수 있다.
- rotate 에 int 값 만큼 시계방향으로 값을 회전한다.(원형리스트 구조)
- extend 에 iterable 객체를 넣으면 요소를 추가한다. extendleft를 하면 순서대로 각 요소를 appendleft 하는 것과 같다.

### OrderedDict
```py
from collections import OrderedDict

od = OrderedDict()
od['a'] = 30
od['d'] = 400
od['c'] = 200
od['b'] = 70

sorted_od = OrderedDict(
    sorted(
        od.items(), 
        key=lambda item: item[0]    # key값 기준
    )
)

des_sorted_od = OrderedDict(
    sorted(
        od.items(),
        key=lambda item: item[1],   # value값 기준
        reverse=True #내림차순
    )
)
```
- dict에 값을 넣은 순서대로 key-value 쌍이 저장되는 것을 보장한다.
- dict 내부의 값을 sorted 적용해서 사용할 때 많이 쓸 것 같다.
- 이후에 나올 Counter도 각 요소의 개수를 dict로 반환하니 잘 사용할 수 있을듯함

## defaultdict
```py
from collections import defaultdict

dd_n = defaultdict(int)
dd_l = defaultdict(list)
dd_f = defaultdict(lambda: 0)   # 0으로 초기화
```
- dict에서 기본값을 설정해줄 수 있다. 그냥 dict 쓰기보다는 defaultdict가 활용성이 훨씬 좋을듯 함

## Counter
```py
from collections import Counter

c = Counter(iterable)
c.update(iterable)

c.most_common()
c.most_common(int)  # ranking

c.subtract(Counter)

Counter + Counter
Counter - Counter
Counter & Counter
Counter | Counter
```
- iterable의 각 요소들 마다 등장하는 횟수를 확인한다.
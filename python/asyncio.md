# 서브루틴과 코루틴
- 서브루틴은 실행 중에 일시정지될 수 없는 녀석이다. 방해받지않고 작업을 끝날때까지 수행한다.
- 코루틴은 실행 중에 일시정지될 수 있다. 그래서 작업 중 정지하고 다른 코루틴에게 실행을 양보할 수 있다.
# 그래서 코루틴을 쓰려면
- async / await 키워드로 간단하게 구현할 수 있다.
- 아래 예시에서 커피를 내리기 시작하고, 5초간 대기한다.
- 5초간 대기하는 동안 토스트를 만들어도 괜찮다. 그래서 sleep(5) 앞에 await를 붙여준다.
- 나는 이제 5초간 기다릴거야~ 그러니까 다른녀석이 일해도 괜찮아~ 의미정도로 받아들이자.

# 메인에서
- 주석 처리한 부분을 살펴보자. 잘못된 사용예시이다.
- async def func()은 함수라기보다는 코루틴 객체라고 보는게 맞다.
- 그래서 코루틴 객체를 작업으로 바꿔주고 실행해야하는 것이다.
- 주석 처리한 부분처럼 실행하면 일반 함수처럼 실행된다. 그래서 동기 처리와 동일하게 동작한다.

```py
import asyncio
import time

def brew_coffee():
    print('Brewing coffee...')
    time.sleep(5)
    print('Coffee is ready!')
    return '☕'

def make_toast():
    print('Making toast...')
    time.sleep(3)
    print('Toast is ready!')
    return '🍞'

def main():
    start_time = time.time()
    coffee = brew_coffee()
    toast = make_toast()
    print('Breakfast is ready!', coffee, toast)

    print('Total time taken:', time.time() - start_time)

async def async_brew_coffee():
    print('Brewing coffee...')
    await asyncio.sleep(5)
    print('Coffee is ready!')
    return '☕'

async def async_make_toast():
    print('Making toast...')
    await asyncio.sleep(3)
    print('Toast is ready!')
    return '🍞'

async def async_main():
    start_time = time.time()

    # coffee = await async_brew_coffee()
    # toast = await async_make_toast()

    coffee_task = asyncio.create_task(async_brew_coffee())
    toast_task = asyncio.create_task(async_make_toast())
    coffee = await coffee_task
    toast = await toast_task
    print('Breakfast is ready!', coffee, toast)

    print('Total time taken:', time.time() - start_time)

if __name__ == '__main__':
    main()
    asyncio.run(async_main())
```

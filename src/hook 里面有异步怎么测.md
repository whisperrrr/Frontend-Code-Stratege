## hook里面有异步怎么测

（这里提到的异步不包含useState）

目前有三种方案:

```
// [强推] 1.act 里面async await一下
// [不推荐] 2.把expect放在setTimeOut里面 ---》 宏任务&微任务
// [是一个迷] 3.waitForNextUpdate
```



### 1.成熟靠谱的``act()``

##### 1.1 这是个什么东西 ?

  [React](https://reactjs.org/docs/test-utils.html#act) 官网是这么说的，我们把render和perform update的代码用act包起来，想当于提供一个接近于React的环境，模拟用户的行为。

  **也就是说，能导致hook里面state发生变化的function行为，都需要用act包起来**。它负责在调用效果后刷新所有效果并重新渲染。

  
##### 1.2 怎么用这个东西 ？

  (1)场景一：function有state变化   

  我们来测试一下用户使用清除按键的行为，

  ```javascript
  const [result, setResult] = useState(0);
    
  const clear = () => {
    setResult(0);
  };
  ```

  测试

  ```javascript
    it("should return zero when act clear", () => {
      const operationEntity = {
        operateNum: 1,
        operatedNum: 2,
        operator: "sum",
      };
      const { useCalculator } = require("../useCalculator");
      const { result } = renderHook(({ operation }) => useCalculator(operation), {
        initialProps: { operation: operationEntity },
      });
  
      expect(result.current.result).toBe(3);
  
      act(() => {
        result.current.clear();
      });
  
      expect(result.current.result).toBe(0);
    });
  
  ```

  （2）场景二：state变化在异步函数的里面 

​      现在有一个场景，我们需要新增一个功能，需要获取到我们的历史计算结果，实现如下：

```javascript
	const [resultHistory, setResultHistory] = useState<string[]>([]);
	const [error, setError] = useState("");

	const getCalculatorHistory = () => {
	 client
	  .get("http://www.example.com/history")
	  .then((response) => {
				setResultHistory(response.data);
		})
	  .catch(() => {
	   setError("something wrong");
	  });
	};
```

​     它的测试是这样的

```javascript
it("should set result list when get result successfully", async () => {
  // Given
  // 坑：这里的api mock不要用nock，否则会出现debug能过，运行不能过得情况
  const get = jest.fn().mockResolvedValue({ data: [1, 2] });
  jest.doMock("src/modules/request/client", () => ({
    client: { get },
  }));

  const operationEntity = {
    operateNum: 1,
    operatedNum: 2,
    operator: "sum",
  };

  const { useCalculator } = require("../useCalculator");
  const { result } = renderHook(({ operation }) => useCalculator(operation), {
    initialProps: { operation: operationEntity },
  });

  expect(result.current.resultHistory).toEqual([]);
  
  // When
  // 重点来了, 在这里等一下 (不管是state还是recoil state都可以用这种方法)
  await act(async () => {
    result.current.getCalculatorHistory();
  });

  // Then
  expect(get).toBeCalledTimes(1);
  expect(result.current.resultHistory).toEqual([1, 2]);
});

```



### 2.谜一般的 ``waitForNextUpdate()``

##### 2.1 这是个什么东西

在react-hook-testing-library里面有很多异步的utils，``waitForNextUpdate`` 返回一个Promise，该Promise在下一次hook渲染的时候resolve。

```javascript
function waitForNextUpdate(options?: { timeout?: number | false }): Promise<void>
```

​     

##### 2.2 等的是个什么东西

```
waitForNextUpdate --> 下一次hook渲染 --> state/recoilState发生变化 
```

```
state/recoilState没有变化 --> 一直等 --> 超时
```

所以说，他等的是hook中的**state(state & recoilState)发生变化**。

##### 2.3 怎么用

还是之前获取历史计算结果的例子，如果用这个api，测试应该是这样

```javascript
  it("should set result list when get result successfully", async () => {
    // Given -- mock api
    const get = jest.fn().mockResolvedValue({ data: [1, 2] });
    jest.doMock("src/modules/request/client", () => ({
      client: { get },
    }));
    // Given -- render hook
    const operationEntity = {
      operateNum: 1,
      operatedNum: 2,
      operator: "sum",
    };
    const { useCalculator } = require("../useCalculator");
    const { result, waitForNextUpdate } = renderHook(({ operation }) => useCalculator(operation), {
      initialProps: { operation: operationEntity },
    });

    // When -- 这里虽然有state变化，但是不用act包起来，因为waitForNextUpdate里面包含了act
    result.current.getCalculatorHistory();
    await waitForNextUpdate();

    // Then 
    expect(get).toBeCalledTimes(1);
    expect(result.current.resultHistory).toEqual([1, 2]);
  });

```



##### 2.4 超时问题 

在我们项目中曾经遇到用``waitForNextUpdate``超时问题，但是不等的话我们得不到异步的结果，测试的expect过不了。  

这里模拟一下当时的场景，还拿我们的``useCalculator``里面的``getCalculatorHistory``举例，

这个时候我们需要有一个recoilState pageLoading，控制全局的loading状态，初始状态是false，当开始getCalculatorHistory的时候，设置为true，执行完毕的时候，将pageLoading的状态，设置为false。

```javascript
// recoil state
export const calculatorLoading = atom<boolean>({
  key: "calculatorLoading",
  default: false
})

```

``useCalculator`` 更改如下：

```javascript
// 将set方法引入  
const setCalculatorLoadingState = useSetRecoilState(calculatorLoading);

// 在get请求开始的时候将loading state设置为true，请求结束后设置为false 
const getCalculatorHistory = () => {
  	setCalculatorLoadingState(true);
  	client
  		.get("http://www.example.com/history")
  		.then((response) => {
      	setResultHistory(response.data);
      	setCalculatorLoadingState(false);
      	console.log(response);
    	})
  		.catch(() => {
 	  		setCalculatorLoadingState(false);
  		});
  };
```

我们还是用``waitForNextUpdate()``的方法来测试一下获取计算结果历史的这个方法，

```javascript
  it("should return the empty array when get history unsuccessfully", async () => {
    // 模拟获取失败的情况
    const get = jest.fn().mockRejectedValue("error");
    jest.doMock("src/modules/request/client", () => ({
      client: { get },
    }));

    const operationEntity = {
      operateNum: 1,
      operatedNum: 2,
      operator: "sum",
    };
    const { useCalculator } = require("../useCalculator");
    
    // 注意这里，因为hook中有recoil state，所以要使用renderRecoilHook，第二个参数可以初始化recoil state
    const { result, waitForNextUpdate } = renderRecoilHook(() => useCalculator(operationEntity), {
      states: [{ recoilState: calculatorLoading, initialValue: false }],
    });

    result.current.getCalculatorHistory();
    await waitForNextUpdate();

    expect(result.current.resultHistory).toEqual([]);
  });

```

Oops，测试挂了，超时了，


这是因为在我们的实现里面，**catch里面的逻辑并没有导致hook里面的state发生变化，所以``waitForNextUpdate``一直在等，从而导致测试超时**，

那么我们现在在hook里面新增一个error的state来记录这个hook的error信息，

```javascript
const [error, setError] = useState("");

const getCalculatorHistory = () => {
	setCalculatorLoadingState(true);
	client
		.get("http://www.example.com/history")
		.then((response) => {
			setResultHistory(response.data);
			setCalculatorLoadingState(false);
		})
		.catch(() => {
			setError("something wrong");
			setCalculatorLoadingState(false);
		});
};
```

然后我们再重新运行测试，过了！破案了！



> **PS**：
>
> 1.如果说，我们的业务场景就是不需要error这个state，那么请使用成熟靠谱的act()来做测试吧
>
> 2.也不要滥用waitForNextUpdate，在非异步的function后面用这个，也是会超时的哦



### 3.其他async的api

##### 3.1 waitForValueChange

这个api也返回一个Promise，当返回的值更改的时候这个Promise就resolve。

依旧是获取计算历史的例子，如果我们只需要关注等到我们需要的值的时候，就可以用这个api，

```javascript
it("should set result list when get result successfully", async () => {
  const get = jest.fn().mockResolvedValue({ data: [1, 2] });
  jest.doMock("src/modules/request/client", () => ({
    client: { get },
  }));

  const operationEntity = {
    operateNum: 1,
    operatedNum: 2,
    operator: "sum",
  };

  const { useCalculator } = require("../useCalculator");

  const { result, waitForValueToChange } = renderHook(({ operation }) => useCalculator(operation), {
    initialProps: { operation: operationEntity },
  });

  expect(result.current.resultHistory).toEqual([]);

  result.current.getCalculatorHistory();
  // 等resultHistory发生改变
  await waitForValueToChange(() => result.current.resultHistory);

  expect(get).toBeCalledTimes(1);
  expect(result.current.resultHistory).toEqual([1, 2]);
});

```



**参考文献：**

[1] [act api的秘密](https://github.com/threepointone/react-act-examples/blob/master/sync.md)

[2] [Fix the "not wrapped in act(...)" warning](https://kentcdodds.com/blog/fix-the-not-wrapped-in-act-warning)

[3] [act源码](https://github.com/facebook/react/blob/master/packages/react-dom/src/test-utils/ReactTestUtilsPublicAct.js)

[4] [react-hooks-testing-library async utilities](https://react-hooks-testing-library.com/reference/api#async-utilities)


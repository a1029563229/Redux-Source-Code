# redux-thunk.js

```es6
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      // 如果 action 为函数，则认定 action 为异步 action
      // 直接执行 action，并将 dispatch, getState, extraArgument 作为入参传入
      // 其中有一个假设，就是假定 action 函数内部会触发 dispatch（异步触发）
      // 此时会再次进入 dispatch 流程，进入中间件处理逻辑，而此时的 action 已经变成了正常的格式（ { type, payload } 格式）
      // 所以 thunk 中间件需要放在 applyMiddleWare 的最前面，这样就可以在接收到异步函数时，阻止后续中间件的继续执行（因为 action 不是一个可识别的 action，并且 dispatch 并没有被同步触发）
      // 所有 redux 的中间件应该坚持一个原则：所有传递给下一个 next 函数的 action，都应该是一个正确格式的 action（ { type, payload } 格式）
      return action(dispatch, getState, extraArgument);
    }

    // 如果 action 为正常 action { type, payload } 格式
    // 则认定为同步 dispatch，则继续执行后续中间件
    return next(action);
  };
}

const thunk = createThunkMiddleware();
thunk.withExtraArgument = createThunkMiddleware;

export default thunk;
```
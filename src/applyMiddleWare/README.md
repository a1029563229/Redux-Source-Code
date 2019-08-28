# applyMiddleWare.js

```es6
import compose from './compose'

/**
 * Creates a store enhancer that applies middleware to the dispatch method
 * of the Redux store. This is handy for a variety of tasks, such as expressing
 * asynchronous actions in a concise manner, or logging every action payload.
 *
 * See `redux-thunk` package as an example of the Redux middleware.
 *
 * Because middleware is potentially asynchronous, this should be the first
 * store enhancer in the composition chain.
 *
 * Note that each middleware will be given the `dispatch` and `getState` functions
 * as named arguments.
 *
 * @param {...Function} middlewares The middleware chain to be applied.
 * @returns {Function} A store enhancer applying the middleware.
 */
// 该函数返回一个 enhancer 增强函数
// 该函数被 createStore 直接调用，并且将 createStore 作为入参传入
export default function applyMiddleware(...middlewares) {
    // ..args 是 reducer 和 preloadedState
    return createStore => (...args) => {
        // 在内部创建了一个 store
        const store = createStore(...args)
        let dispatch = () => {
            throw new Error(
                'Dispatching while constructing your middleware is not allowed. ' +
                'Other middleware would not be applied to this dispatch.'
            )
        }

        // 创建了一个 middlewareAPI
        const middlewareAPI = {
            getState: store.getState,
            dispatch: (...args) => dispatch(...args)
        }

        // 将中间件依次调用，将 { getState, dispatch } 作为入参传入，并返回一个新的函数调用链
        const chain = middlewares.map(middleware => middleware(middlewareAPI))
            // 最后将 dispatch 作为入参传入
        dispatch = compose(...chain)(store.dispatch)

        // 该返回值是 createStore 中包含 enhancer 时的返回值
        // 返回了 store，并且将转换过的 dispatch 覆盖原 dispatch 值
        return {
            ...store,
            dispatch
        }
    }
}
```
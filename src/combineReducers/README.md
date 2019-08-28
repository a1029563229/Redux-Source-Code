# combineReducers.js

```es6
import ActionTypes from './utils/actionTypes'
import warning from './utils/warning'
import isPlainObject from './utils/isPlainObject'

// 对 return undefined 的 reducer 进行报错处理
function getUndefinedStateErrorMessage(key, action) {
    const actionType = action && action.type
    const actionDescription =
        (actionType && `action "${String(actionType)}"`) || 'an action'

    return (
        `Given ${actionDescription}, reducer "${key}" returned undefined. ` +
        `To ignore an action, you must explicitly return the previous state. ` +
        `If you want this reducer to hold no value, you can return null instead of undefined.`
    )
}

function getUnexpectedStateShapeWarningMessage(
    inputState,
    reducers,
    action,
    unexpectedKeyCache
) {
    const reducerKeys = Object.keys(reducers)
    const argumentName =
        action && action.type === ActionTypes.INIT ?
        'preloadedState argument passed to createStore' :
        'previous state received by the reducer'

    // reducer 不能为空
    if (reducerKeys.length === 0) {
        return (
            'Store does not have a valid reducer. Make sure the argument passed ' +
            'to combineReducers is an object whose values are reducers.'
        )
    }

    // state 需要是一个纯 Object
    if (!isPlainObject(inputState)) {
        return (
            `The ${argumentName} has unexpected type of "` + {}.toString.call(inputState).match(/\s([a-z|A-Z]+)/)[1] +
            `". Expected argument to be an object with the following ` +
            `keys: "${reducerKeys.join('", "')}"`
        )
    }

    const unexpectedKeys = Object.keys(inputState).filter(
        key => !reducers.hasOwnProperty(key) && !unexpectedKeyCache[key]
    )

    unexpectedKeys.forEach(key => {
        unexpectedKeyCache[key] = true
    })

    if (action && action.type === ActionTypes.REPLACE) return

    if (unexpectedKeys.length > 0) {
        return (
            `Unexpected ${unexpectedKeys.length > 1 ? 'keys' : 'key'} ` +
            `"${unexpectedKeys.join('", "')}" found in ${argumentName}. ` +
            `Expected to find one of the known reducer keys instead: ` +
            `"${reducerKeys.join('", "')}". Unexpected keys will be ignored.`
        )
    }
}

function assertReducerShape(reducers) {
    Object.keys(reducers).forEach(key => {
        const reducer = reducers[key]
            // 第一个参数为 undefined 时，reducer 将会取默认值（比如 state = initialState）
        const initialState = reducer(undefined, { type: ActionTypes.INIT })

        if (typeof initialState === 'undefined') {
            throw new Error(
                `Reducer "${key}" returned undefined during initialization. ` +
                `If the state passed to the reducer is undefined, you must ` +
                `explicitly return the initial state. The initial state may ` +
                `not be undefined. If you don't want to set a value for this reducer, ` +
                `you can use null instead of undefined.`
            )
        }

        if (
            typeof reducer(undefined, {
                type: ActionTypes.PROBE_UNKNOWN_ACTION()
            }) === 'undefined'
        ) {
            throw new Error(
                `Reducer "${key}" returned undefined when probed with a random type. ` +
                `Don't try to handle ${ActionTypes.INIT} or other actions in "redux/*" ` +
                `namespace. They are considered private. Instead, you must return the ` +
                `current state for any unknown actions, unless it is undefined, ` +
                `in which case you must return the initial state, regardless of the ` +
                `action type. The initial state may not be undefined, but can be null.`
            )
        }
    })
}

/**
 * Turns an object whose values are different reducer functions, into a single
 * reducer function. It will call every child reducer, and gather their results
 * into a single state object, whose keys correspond to the keys of the passed
 * reducer functions.
 *
 * @param {Object} reducers An object whose values correspond to different
 * reducer functions that need to be combined into one. One handy way to obtain
 * it is to use ES6 `import * as reducers` syntax. The reducers may never return
 * undefined for any action. Instead, they should return their initial state
 * if the state passed to them was undefined, and the current state for any
 * unrecognized action.
 *
 * @returns {Function} A reducer function that invokes every reducer inside the
 * passed object, and builds a state object with the same shape.
 */
export default function combineReducers(reducers) {
    const reducerKeys = Object.keys(reducers)
    const finalReducers = {}
    for (let i = 0; i < reducerKeys.length; i++) {
        const key = reducerKeys[i]

        if (process.env.NODE_ENV !== 'production') {
            if (typeof reducers[key] === 'undefined') {
                warning(`No reducer provided for key "${key}"`)
            }
        }

        if (typeof reducers[key] === 'function') {
            finalReducers[key] = reducers[key]
        }
    }
    const finalReducerKeys = Object.keys(finalReducers)

    // This is used to make sure we don't warn about the same
    // keys multiple times.
    let unexpectedKeyCache
    if (process.env.NODE_ENV !== 'production') {
        unexpectedKeyCache = {}
    }

    let shapeAssertionError
    try {
        assertReducerShape(finalReducers)
    } catch (e) {
        shapeAssertionError = e
    }

    // 第一个入参为 createStore 内部的 currentState
    return function combination(state = {}, action) {
        if (shapeAssertionError) {
            throw shapeAssertionError
        }

        if (process.env.NODE_ENV !== 'production') {
            const warningMessage = getUnexpectedStateShapeWarningMessage(
                state,
                finalReducers,
                action,
                unexpectedKeyCache
            )
            if (warningMessage) {
                warning(warningMessage)
            }
        }

        let hasChanged = false
        const nextState = {}
        for (let i = 0; i < finalReducerKeys.length; i++) {
            const key = finalReducerKeys[i]
            const reducer = finalReducers[key]
            const previousStateForKey = state[key]
            const nextStateForKey = reducer(previousStateForKey, action)
            if (typeof nextStateForKey === 'undefined') {
                const errorMessage = getUndefinedStateErrorMessage(key, action)
                throw new Error(errorMessage)
            }
            nextState[key] = nextStateForKey
            // 执行 reducer 后，判断 state 是否被 reducer 改变
            // 由于使用的是 !== 符号进行对比，所以只修改原有对象，而不返回一个新的对象是无法触发 hasChanged 这个检查的
            hasChanged = hasChanged || nextStateForKey !== previousStateForKey
        }
        hasChanged =
            hasChanged || finalReducerKeys.length !== Object.keys(state).length

        // 如果 store 中有多个 reducer，那么 state 就是这些 reducer key 组成的一个复合对象
        return hasChanged ? nextState : state
    }
}
```
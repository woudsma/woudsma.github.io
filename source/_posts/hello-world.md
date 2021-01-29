---
title: Hello world
---

## Test

```js
console.log('test')

const a = () => {
  
}
```

### And another one

```js
/**
 * Redux form store factory function
 *
 * @returns {object} store
 */
export const createFormStore = preloadedState => createStore(reducer, preloadedState);

/**
 * Rarely used hook for retrieving the form store directly.
 * Please try to use useFormSelector to access store values.
 *
 * @returns {function} useFormStore
 */
export const useFormStore = createStoreHook(FormContext);

/**
 * Form dispatch hook, similar to react-redux's useDispatch.
 * Actions dispatched using this hook will only affect the current form context.
 *
 * @returns {function} useFormDispatch
 */
export const useFormDispatch = createDispatchHook(FormContext);

/**
 * Form selector hook, similar to react-redux's useSelector.
 * Use this hook to retrieve data from the form store.
 *
 * @returns {function} useFormSelector
 */
export const useFormSelector = createSelectorHook(FormContext);
```

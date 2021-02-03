---
title: How we reduced our re-renders by 80% with React Redux and custom context
---

## Introduction

> _DRAFT_

### Your app currently looks something like this

<!--  title="App.jsx" -->

```js
import React from "react";
import { render } from "react-dom";
import { Provider } from "react-redux";

import store from "./globalStore";

import App from "./App";

render(
  <Provider store={store}>
    <App />
  </Provider>,
  document.getElementById("root"),
);
```

### With a 'global' Redux store already in place

```js
import { createStore } from "redux";

/**
 * Define and export action types.
 */
export const SOME_GLOBAL_FORM_ACTION = "SOME_GLOBAL_FORM_ACTION";
export const SOME_GLOBAL_RESET_FORM_ACTION = "SOME_GLOBAL_RESET_FORM_ACTION";

/**
 * Example 'global' redux store
 * that might exist in your app already.
 */
const initialState = {
  forms: {},
  someUserData: {},
  someOtherData: {},
};

/**
 * A standard reducer function.
 *
 * @param {object} state
 * @param {object} action
 * @returns {object} nextState
 */
const reducer = (state = initialState, { type, payload }) => {
  switch (type) {
    case SOME_GLOBAL_FORM_ACTION:
      return {
        ...state,
        forms: { ...state.forms, [payload.formId]: payload.values },
      };
    case SOME_GLOBAL_RESET_FORM_ACTION:
      return {
        ...state,
        forms: {
          ...state.forms,
          [payload.formId]: initialState.forms?.[formId],
        },
      };
    // ... etc
    default:
      return state;
  }
};

export default createStore(reducer);
```

### We've got several instances of large / complex components in our app

```js
import React from "react";

import { FormContainer } from "./FormContainer";

/**
 * App component with some example form components.
 *
 * @returns {React.FC}
 */
const App = () => (
  <div className="App">
    <h1>React + Redux app</h1>
    <hr />
    <FormContainer formId="homeBillingForm" />
    <FormContainer formId="homeShippingForm" />
    <FormContainer formId="guestLoginForm" />
  </div>
);

export default App;
```

### Seperate logic from presentation with container and presentational components

Presentational components can be tested with ease. Simply mock the required props and you're good to go.
You should be able to write these components without the `return` keyword, as in the following example.
I try to do no logic at all inside presentational components.

```js
import React from "react";

/**
 * Form presentational component.
 *
 * @returns {React.FC}
 */
const Form = ({ children, ...props }) => <form {...props}>{children}</form>;

export default Form;
```

```js
import React from "react";

/**
 * Example input presentational component.
 *
 * @returns {React.FC}
 */
const FormInput = ({ children, ...props }) => (
  <div className="FormInput">
    <input type="text" {...props} />
    {children}
  </div>
);

export default FormInput;
```

### To avoid prop-drilling, we'd like to get values using [React Context](https://reactjs.org/docs/context.html)

Simply create an empty context, we will use this later on.

```js
import { createContext } from "react";

const FormContext = createContext(null);

export default FormContext;
```

### Create the container components

```js
import React, { useCallback, useEffect, useRef } from "react";
import { Provider, useDispatch } from "react-redux";

import { SOME_GLOBAL_FORM_ACTION } from "./globalStore";
import FormContext from "./context";
import { createFormStore, useFormActions, useFormSelector } from "./hooks";
import { valuesSelector, requiredFieldsFilledSelector } from "./selectors";

import Form from "./Form";

/**
 * Form container component.
 * In this component we will handle business logic.
 *
 * @returns {React.FC}
 */
const FormContainer = ({ formId, children, ...props }) => {
  /**
   * We're free to use the 'global' dispatch function next to
   * the form dispatch function by using useDispatch and useFormDispatch.
   */
  const dispatch = useDispatch();

  const { setFormId } = useFormActions();

  /**
   * Get values from the form store using our
   * custom redux hook 'useFormSelector'.
   */
  const values = useFormSelector(valuesSelector);
  const requiredFieldsFilled = useFormSelector(requiredFieldsFilledSelector);

  useEffect(() => {
    setFormId(formId);
  }, [formId]);

  const onSubmitHandler = useCallback(
    (e) => {
      e.preventDefault();

      if (requiredFieldsFilled) {
        dispatch({
          type: SOME_GLOBAL_FORM_ACTION,
          payload: { formId, values },
        });
      }
    },
    [dispatch, requiredFieldsFilled, formId, values],
  );

  return (
    <Form onSubmit={onSubmitHandler} {...props}>
      {children}
    </Form>
  );
};

/**
 * Connected form container component.
 * Connects the component to the form redux store using a custom context.
 * This allow the use of custom redux hooks.
 * https://react-redux.js.org/next/api/hooks#custom-context
 */
const ConnectedFormContainer = connect(null, null, null, {
  context: FormContext,
})(FormContainer);

/**
 * Form container provider.
 * Provides a redux store for each form instance and renders the connected form container.
 * https://react-redux.js.org/using-react-redux/accessing-store#providing-custom-context
 *
 * You can provide an optional parameter 'store' which can be useful when writing tests.
 * (not covered in this blog post)
 *
 * @returns {React.FC}
 */
const FormContainerProvider = ({
  store = createFormStore(),
  children,
  ...props
}) => {
  /**
   * Create a new store for every instance of FormContainerProvider.
   * 'useRef' makes sure we get a consistent reference to the store object.
   */
  const { current: formStore } = useRef(store);

  return (
    <Provider context={FormContext} store={formStore}>
      <ConnectedFormContainer {...props}>{children}</ConnectedFormContainer>
    </Provider>
  );
};

/**
 * Export the form container provider instead of the form container component.
 * I'm renaming the wrapper back to FormContainer in the export, but that's optional.
 */
export { FormContainerProvider as FormContainer };
```

### Retrieving form values and firing actions from a deeply nested component

It doesn't matter how deep this component is nested in the `FormContainerProvider`.
Calling `setValue` will update a value in the current store context only.
No need to use `connect` and `mapStateToProps` / `mapDispatchToProps`. :)

```js
import React from "react";

import { useFormActions } from "./hooks";

import FormInput from "./FormInput";

/**
 * Form input container component.
 *
 * @returns {React.FC}
 */
const FormInputContainer = ({ name, children, ...props }) => {
  /**
   * ❌ creates a subscription and triggers a re-render on all updates to FormContext.
   */
  // const { someValue } = useContext(FormContext);

  /**
   * ⚠️ creates a subscription to a single store value.
   */
  // const someValue = useFormSelector(state => state.someValue);

  /**
   * ✅ creates a memoized subscription to a single store value.
   */
  // const someValue = useFormSelector(someValueSelector);

  const { setValue } = useFormActions();

  const onChangeHandler = useCallback(
    (e) => {
      setValue(name, e.target.value);
    },
    [setValue, name],
  );

  return (
    <FormInput onChange={onChangeHandler} {...props}>
      {children}
    </FormInput>
  );
};

export default FormInputContainer;
```

### Setting up some example complex logic of our form

We would need to validate input, write things to localStorage - but not when it's a password - etc.

```js
/**
 * Just an example utility function that we would use in our complex form.
 *
 * @param {string} value An input value
 * @param {object} validationRule A dynamic validation rule
 * @returns {boolean} isValid
 */
export const validateInput = (value, validationRule) => {
  if (value?.length > validationRule?.maxLength) {
    return false;
  }

  if (!value && validationRule?.isRequired) {
    return false;
  }

  return true;
};
```

### This is where the magic happens

**Hooks.**

React Redux exposes three useful functions which we can use to create custom redux hooks.

- `createStoreHook`
- `createDispatchHook`
- `createSelectorHook`

If we pass a context into these functions, we can create custom `useDispatch` and `useSelector` functions, that only operate on the 'local' / 'sub' store, the form store in our case.
In this post I'm creating `useFormSelector` for example, but if you're creating complex modals you could create `useModalDispatch` and `useModalSelector` for example - and use them if you have a context + store set up as shown above in `FormContainer.jsx`.

```js
import { useCallback } from "react";
import { createStore } from "redux";
import {
  createStoreHook,
  createDispatchHook,
  createSelectorHook,
  useDispatch,
} from "react-redux";

import FormContext from "./context";
import { SOME_GLOBAL_RESET_FORM_ACTION } from "../globalStore";
import { formIdSelector, validationRulesSelector } from "./selectors";
import { validateInput } from "./utils";

/**
 * Form reducer constants.
 */
const SET_FORM_ID = "SET_FORM_ID";
const SET_VALUE = "SET_VALUE";
const SET_VALUE = "SET_VALIDITY_VALUE";
const SET_VALIDATION_RULES = "SET_VALIDATION_RULES";
const RESET_FORM_STATE = "RESET_FORM_STATE";

/**
 * Form reducer initial state.
 */
const initialState = {
  formId: undefined,
  values: {},
  validityValues: {},
  validationRules: {},
};

/**
 * Another standard reducer function.
 * This reducer will only handle form actions.
 *
 * @param {object} state
 * @param {object} action
 * @returns {object} nextState
 */
const reducer = (state = initialState, { type, payload }) => {
  switch (type) {
    case SET_FORM_ID:
      return { ...state, formId: payload };
    case SET_VALUE:
      return {
        ...state,
        values: { ...state.values, [payload.key]: payload.value },
      };
    case SET_VALIDITY_VALUE:
      return {
        ...state,
        validityValues: { ...state.validityValues, [payload.key]: payload.value },
      };
    case SET_VALIDATION_RULES:
      return {
        ...state,
        validationRules: { ...state.validationRules, ...payload },
      };
    case RESET_FORM_STATE:
      return initialState;
    default:
      return state;
  }
};

/**
 * Redux form store factory function.
 *
 * Note that 'preloadedState' is not used in this example.
 * It's useful when you want to provide a custom initial state
 * when writing tests for example.
 *
 * @returns {object} store
 */
export const createFormStore = (preloadedState) =>
  createStore(reducer, preloadedState);

/**
 * Rarely used hook for retrieving the form store directly.
 * Preferably, use useFormSelector to access store values.
 */
export const useFormStore = createStoreHook(FormContext);

/**
 * Form dispatch hook, similar to react-redux's useDispatch hook.
 * Actions dispatched using this hook will only affect the specified context.
 */
export const useFormDispatch = createDispatchHook(FormContext);

/**
 * Form selector hook, similar to react-redux's useSelector.
 * Use this hook to retrieve data from the form store.
 */
export const useFormSelector = createSelectorHook(FormContext);

/**
 * Hook for convenient access to the form redux actions.
 *
 * @returns {object} formActions
 */
export const useFormActions = () => {
  /**
   * Use useDispatch and useFormDispatch to be able to
   * dispatch actions to both the form store and the global store.
   */
  const dispatch = useDispatch();
  const formDispatch = useFormDispatch();

  /**
   * Get (aka select) some values from the form store with 'useFormSelector'.
   * It's no problem to use these hooks inside other hooks like this.
   */
  const formId = useFormSelector(formIdSelector);
  const validationRules = useFormSelector(validationRulesSelector);

  /**
   * Sets the form id.
   *
   * @param {string} id
   */
  const setFormId = useCallback(
    (id) => formDispatch({ type: SET_FORM_ID, payload: id }),
    [formDispatch],
  );

  /**
   * Sets a form value and does a validation check.
   * We keep track of the value's validity using the 'validityValues' object.
   *
   * @param {string} key
   * @param {string} value
   */
  const setValue = useCallback(
    (key, value) => {
      if (value === undefined) return;

      formDispatch({
        type: SET_VALUE,
        payload: { key, value },
      })

      const isValid = validateInput(value, validationRules?.[key])

      formDispatch({
        type: SET_VALIDITY_VALUE,
        payload: { key, value: isValid },
      })
    }
    [formDispatch, validateInput, validationRules],
  );

  /**
   * Sets the validation rules.
   *
   * @param {object} validationRules
   */
  const setValidationRules = useCallback(
    (validationRules) => {
      formDispatch({ type: SET_VALIDATION_RULES, payload: validationRules })
    },
    [formDispatch],
  );

  /**
   * Reset the entire form state in the current context.
   * And - as an example - also update the global redux store.
   */
  const resetFormValues = useCallback(() => {
    formDispatch({ type: RESET_FORM_STATE });
    dispatch({ type: SOME_GLOBAL_RESET_FORM_ACTION, payload: formId });
  }, [formDispatch, dispatch, formId]);

  return {
    setFormId,
    setValue,
    setValidationRules,
    resetFormValues,
  };
};
```

### The cherry on top - [reselect](https://github.com/redux/reselect)

Organize and create your store value selectors in a central location to be re-used across multiple components.
A great advantage of writing selectors like this is that each selector is a 'pure' function. Which is awesome insurance against side-effects.

```js
import isEqual from "react-fast-compare";
import {
  createSelector,
  createSelectorCreator,
  defaultMemoize,
} from "reselect";

/**
 * Creates a custom selector creator function.
 * The resulting selector performs a deep equality comparison.
 * Uses the 'isEqual' function from 'react-fast-compare'
 * Useful for memoization of values of type object or array.
 *
 * The default 'createSelector' performs simple equality comparison with '==='.
 */
export const createDeepEqualSelector = createSelectorCreator(
  defaultMemoize,
  isEqual,
);

/**
 * Composable memoized redux selector functions.
 * Syntax: createSelector|createDeepEqualSelector(...inputSelectors, resultFn)
 * https://github.com/reduxjs/reselect
 *
 * Each selector must be a 'pure' function.
 * A benefit of this is that it makes selectors very reliable and easily testable.
 */
export const formIdSelector = createSelector(
  (state) => state?.formId,
  (formId) => formId,
);

export const valuesSelector = createDeepEqualSelector(
  (state) => state?.values || {},
  (values) => values,
);

export const validationRulesSelector = createDeepEqualSelector(
  (state) => state?.validationRules || {},
  (validationRules) => validationRules,
);

export const validityValuesSelector = createDeepEqualSelector(
  (state) => state?.validityValues || {},
  (validityValues) => validityValues,
);

/**
 * Example of what we used to do inside components, but is a
 * great candidate to move to a re-usable selector, like so:
 */
export const requiredFieldsFilledSelector = createSelector(
  validationRulesSelector,
  validityValuesSelector,
  (validationRules, validityValues) =>
    Object.keys(validationRules)
      .filter((key) => validationRules?.[key]?.required)
      .every((key) => validityValues?.[key] === true),
);
```

### Combining selectors and using arguments

You can imagine that a form store contains many properties, some of which you want to combine to create new - more specific - values.
Some components don't need a subscription to the entire `formConfigs` object in the following example snippet.
It could be a large object that some components actually do need to subscribe to, but others might only need a subset of the data from that object.
Generally it's a good idea to move as much logic to selectors when combining values from the store.

Writing lots of selectors might seem stupid at first, but when selecting a value like `state.cart.cart.consumer.billingAddress.street` in a few components, there's a high chance to make a small mistake and write `state.cart.consumer.billingAddress.street` instead. Defining and having things in a centralised place prevents accidental typo's and gives the benefit of being able to make a single change to update all affected components.

```js
export const formConfigsSelector = createDeepEqualSelector(
  (state) => state?.formConfigs || {},
  (formConfigs) => formConfigs,
);

export const useShippingAsBillingSelector = createSelector(
  (state) => state?.useShippingAsBilling,
  (useShippingAsBilling) => useShippingAsBilling,
);

/**
 * Example of an polymorphic selector creator.
 * (selector which you can pass an argument)
 */
export const createFormConfigSelector = (formId) =>
  createDeepEqualSelector(
    formConfigsSelector,
    (formConfigs) => formConfigs?.[formId] || {},
  );

/**
 * Example usage of an polymorphic selector.
 */
export const formConfigShippingSelector = createDeepEqualSelector(
  createFormConfigSelector("homeShippingForm"),
  (formConfigShipping) => formConfigShipping,
);

/**
 * This selector selects a value that has slightly different behavior
 * as is often the case in more complex apps.
 */
export const formConfigBillingSelector = createDeepEqualSelector(
  createFormConfigSelector("homeBillingForm"),
  useShippingAsBillingSelector,
  (formConfigBilling, useShippingAsBilling) =>
    useShippingAsBilling ? {} : formConfigBilling,
);

export const combinedFormConfigSelector = createDeepEqualSelector(
  formConfigShippingSelector,
  formConfigBillingSelector,
  (formConfigShipping, formConfigBilling) => ({
    ...formConfigShipping,
    ...formConfigBilling,
  }),
);

/**
 * Or:
 */
export const combinedFormConfigSelector = createDeepEqualSelector(
  formConfigShippingSelector,
  formConfigBillingSelector,
  (...args) => Object.assign({}, ...args),
);

/**
 * Transforming the selection even further:
 */
export const combinedFormConfigValuesSelector = createDeepEqualSelector(
  combinedFormConfigSelector,
  Object.values,
);
```

I hope these tricks can benefit you in your projects as much as it did in our case!

If you found this interesting, consider following me on [Github](https://github.com/woudsma). I'm very much open to questions/feedback, don't hesitate to send a message or leave a reply ;)

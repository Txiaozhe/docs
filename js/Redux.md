* app

  * actions
    * actionTypes.js 事件类型
    * counterActions.js 事件名称，返回事件类型
  * components
    * counter.js  组件
  * containers
    * app.js  入口
    * counterApp.js 绑定状态
  * reducers
    * counter.js 实现事件
    * index.js 导出事件

* app/actions/actionTypes.js

  ```javascript
  export const INCREMENT = 'INCREMENT';
  export const DECREMENT = 'DECREMENT';
  ```

* app/actions/counterActions.js

  ```javascript
  import * as types from './actionTypes';

  export function increment() {
    return {
      type: types.INCREMENT
    };
  }

  export function decrement() {
    return {
      type: types.DECREMENT
    };
  }
  ```

* app/components/counter.js

  ```javascript
  import React, {Component} from 'react';
  import {StyleSheet, View, Text, TouchableOpacity} from 'react-native';

  const styles = StyleSheet.create({
    button: {
      width: 100,
      height: 30,
      padding: 10,
      backgroundColor: 'lightgray',
      alignItems: 'center',
      justifyContent: 'center',
      margin: 3
    }
  });

  export default class Counter extends Component {
    constructor(props) {
      super(props);
    }

    render() {
        //const { counter, increment, decrement } = this.props;
        const counter = this.props.counter;
        const increment = this.props.increment;
        const decrement = this.props.decrement;

      return (
        <View style={{flex: 1, alignItems: 'center', justifyContent: 'center' }}>
          <Text>{counter}</Text>
          <TouchableOpacity onPress={increment} style={styles.button}>
            <Text>up</Text>
          </TouchableOpacity>
          <TouchableOpacity onPress={decrement} style={styles.button}>
            <Text>down</Text>
          </TouchableOpacity>
        </View>
      );
    }
  }
  ```

* app/containers/app.js

  ```javascript
  import React, {Component} from 'react';
  import { createStore, applyMiddleware, combineReducers } from 'redux';
  import { Provider } from 'react-redux';
  import thunk from 'redux-thunk';

  import * as reducers from '../reducers';
  import CounterApp from './counterApp';

  const createStoreWithMiddleware = applyMiddleware(thunk)(createStore);
  const reducer = combineReducers(reducers);
  const store = createStoreWithMiddleware(reducer);

  export default class App extends Component {
    render() {
      return (
        <Provider store={store}>
          <CounterApp />
        </Provider>
      );
    }
  }
  ```

* app/containers/counterApp.js

  ```javascript
  'use strict';

  import React, {Component} from 'react';
  import {bindActionCreators} from 'redux';
  import Counter from '../components/counter';
  import * as counterActions from '../actions/counterActions';
  import { connect } from 'react-redux';

  class CounterApp extends Component {
    constructor(props) {
      super(props);
    }
    //{...actions}
    render() {
      const { state, actions } = this.props;
      return (
        <Counter
          counter={state.count}
          increment={actions.increment}
          decrement={actions.decrement}/>
      );
    }
  }

  export default connect(state => ({
      state: state.counter
    }),
    (dispatch) => ({
      actions: bindActionCreators(counterActions, dispatch)
    })
  )(CounterApp);
  ```

* app/reducers/counter.js

  ```javascript
  import * as types from '../actions/actionTypes';

  const initialState = {
    count: 0
  };

  export default function counter(state = initialState, action = {}) {
    switch (action.type) {
      case types.INCREMENT:
        return {
          ...state,
          count: state.count + 1
        };
      case types.DECREMENT:
        return {
          ...state,
          count: state.count - 1
        };
      default:
        return state;
    }
  }
  ```

* app/reducers/index.js

  ```javascript
  import counter from './counter';

  export {
    counter
  };
  ```

  ​
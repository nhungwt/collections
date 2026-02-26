# **1. Redux Store là gì và có vai trò gì trong ứng dụng React?**

__Đúng ✅:__

- Redux Store lưu trữ state cho cả ứng dụng
- Dễ truy xuất từ bất kỳ component nào mà không cần truyền qua props

__Còn thiếu ⚠️:__ Redux Store còn làm nhiều việc khác:

- Chứa __reducers__ (các hàm xử lý actions để update state)

- Có các methods chính:

  - `getState()` - lấy state hiện tại
  - `dispatch(action)` - gửi action để update state
  - `subscribe(listener)` - đăng ký lắng nghe thay đổi state

- Chứa __middleware__ (như redux-thunk, redux-logger) để xử lý logic bổ sung

- Chỉ có __MỘT store duy nhất__ cho toàn bộ ứng dụng

__Ví dụ:__

```typescript
import { configureStore } from '@reduxjs/toolkit';

const store = configureStore({
  reducer: {
    todos: todosReducer,
    users: usersReducer,
  },
  middleware: (getDefaultMiddleware) => 
    getDefaultMiddleware().concat(logger),  // Thêm middleware
});

// Store có các methods:
console.log(store.getState());  // ← Lấy state
store.dispatch(addTodo('Learn Redux'));  // ← Dispatch action
store.subscribe(() => {
  console.log('State changed:', store.getState());  // ← Lắng nghe thay đổi
});
```

# **2. `createSlice` làm gì và trả về những gì?**

__SAI ❌:__ Bạn đang nhầm lẫn giữa `createSlice` và `createSelector`!

### `createSlice` (đúng):

```typescript
import { createSlice } from '@reduxjs/toolkit';

const todosSlice = createSlice({
  name: 'todos',                    // ← Tên của slice
  initialState: [],                  // ← State ban đầu
  reducers: {                        // ← Định nghĩa sync actions
    addTodo: (state, action) => {
      state.push(action.payload);
    },
    removeTodo: (state, action) => {
      return state.filter(todo => todo.id !== action.payload);
    },
  },
  extraReducers: (builder) => {      // ← Định nghĩa async actions
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.loading = false;
        state.data = action.payload;
      });
  },
});

// createSlice TRẢ VỀ:
const { actions, reducer, selectors } = todosSlice;
// actions = { addTodo, removeTodo }
// reducer = function xử lý state
// selectors = { các selectors của slice }

export const { addTodo, removeTodo } = todosSlice.actions;
export default todosSlice.reducer;
```

__Tóm tắt `createSlice`:__

- __Nhận:__ Object `{ name, initialState, reducers, extraReducers }`
- __Trả về:__ Object `{ actions, reducer, selectors, ... }`
- __Tự động tạo:__ Actions từ reducers

### `createSelector` (bạn mô tả):

```typescript
import { createSelector } from '@reduxjs/toolkit';

const selectAllTodos = (state) => state.todos;

const selectCompletedTodos = createSelector(
  [selectAllTodos, (state, userId) => userId],  // ← Input selectors
  (todos, userId) => {                           // ← Output function
    return todos.filter(todo => todo.completed && todo.userId === userId);
  }
);
```

__Tóm tắt `createSelector`:__

- __Nhận:__ Array các input selectors + output function
- __Trả về:__ Memoized selector function
- __Tự động:__ Re-calculate khi input thay đổi


# **3. Khác nhau giữa `reducers` và `extraReducers` trong `createSlice` là gì? Khi nào dùng cái nào?**

- reducers = actions nội bộ, sync
- extraReducers = actions từ thunks/API hoặc slices khác

__Quy tắc:__ Nếu action đó được `dispatch()` và bạn **muốn react** khi action đó xảy ra → Dùng `extraReducers`. Ngược lại → KHÔNG dùng.


# **4. `createAsyncThunk` là gì và nó tạo ra những action types nào?**

__Quy tắc:__

- Redux chỉ chấp nhận __plain object actions__ (sync)
- Để làm async, cần dùng __thunk__ - function trả về function
- `createAsyncThunk` tạo ra thunk có thể dispatch 3 actions (pending, fulfilled, rejected)
- `createAsyncThunk` tự động wrap async function trong `try-catch`



### Chi tiết về 3 action types

```typescript
const fetchTodos = createAsyncThunk('todos/fetch', async (payload) => {
  const response = await fetch('/api/todos');
  return response.json();  // ← Data sẽ có trong action.payload
});

// createAsyncThunk TỰ ĐỘNG tạo 3 actions:

// 1. pending action (dispatch khi bắt đầu gọi API)
{
  type: 'todos/fetch/pending',
  meta: {
    requestId: 'unique-id',      // ← ID của request
    arg: { /* payload ban đầu */ } // ← Tham số truyền vào
  }
}

// 2. fulfilled action (dispatch khi API thành công - status 2xx)
{
  type: 'todos/fetch/fulfilled',
  payload: [ /* data từ API */ ],  // ← Kết quả từ return trong async function
  meta: {
    requestId: 'unique-id',
    arg: { /* payload ban đầu */ }
  }
}

// 3. rejected action (dispatch khi API thất bại)
{
  type: 'todos/fetch/rejected',
  error: {
    message: 'Failed to fetch',    // ← Error message
    code: '404'                     // ← Error code
  },
  meta: {
    requestId: 'unique-id',
    arg: { /* payload ban đầu */ }
  }
}
```
```
.addCase(fetchUsers.rejected, (state, action) => {
  // action.payload = error từ rejectWithValue
  // action.error = error mặc định từ createAsyncThunk
  
  if (action.payload) {
    // Nếu dùng rejectWithValue
    console.log(action.payload.message);  // ← "User not found"
    console.log(action.payload.code);      // ← 404
  } else {
    // Nếu throw error hoặc Promise.reject
    console.log(action.error.message);
  }
});

```

### Ví dụ thực tế:

```typescript
// Tạo async thunk
const fetchUsers = createAsyncThunk('users/fetch', async (userId) => {
  const response = await fetch(`/api/users/${userId}`);
  
  if (!response.ok) {
    throw new Error('Failed to fetch user');  // → dispatch rejected
  }
  
  const userData = await response.json();
  return userData;  // → dispatch fulfilled với payload = userData
});
```

![alt text](image.png)

---------------
"Thunk là function, có thể dispatch nhiều actions"

thunk là function thì ok, nhưng "có thể dispatch nhiều actions" nghĩa là sao, chẳng lẽ duntion nào cũng có thể dispatch hả

Câu hỏi rất chính xác! Tôi xin lỗi vì nói chưa rõ. **KHÔNG** - không phải function nào cũng có thể dispatch!

## Giải thích rõ ràng:

### Normal function - KHÔNG thể dispatch:

```typescript
// ❌ SAi: Function thường KHÔNG thể dispatch
function myFunction() {
  dispatch({ type: 'SOME_ACTION' });  // ← Lỗi! dispatch không tồn tại
}

// ❌ SAi: Redux không nhận function làm action
dispatch(myFunction());  // ← Lỗi! Action phải là plain object, không phải function
```

### Redux Thunk - function đặc biệt CÓ THỂ dispatch:

```typescript
// ✅ Đúng: Thunk (với redux-thunk middleware) CÓ THỂ dispatch
const myThunk = createAsyncThunk('myThunk', async (payload, { dispatch }) => {
  // ← Nhận dispatch từ tham số thứ 2
  
  dispatch({ type: 'ACTION_1' });     // ← Dispatch action 1
  dispatch({ type: 'ACTION_2' });     // ← Dispatch action 2
  dispatch({ type: 'ACTION_3' });     // ← Dispatch action 3
});
```

## Tại sao cần redux-thunk middleware?

### Redux mặc định - Chỉ nhận plain object actions:

```typescript
// Redux Store mặc định
const store = createStore(reducer);

// ✅ Đúng: Plain object action
store.dispatch({
  type: 'ADD_TODO',
  payload: { text: 'Learn Redux' }
});

// ❌ SAI: Function action (không có middleware)
store.dispatch(function() {
  // Redux sẽ báo lỗi: "Actions must be plain objects"
});
```

### Redux với redux-thunk middleware - Có thể dispatch function:

```typescript
import { configureStore } from '@reduxjs/toolkit';
// hoặc import thunk from 'redux-thunk';

const store = configureStore({
  reducer: rootReducer,
  // redux-thunk middleware được tự động thêm
});

// ✅ Đúng: Plain object action
store.dispatch({
  type: 'ADD_TODO',
  payload: { text: 'Learn Redux' }
});

// ✅ Đúng: Function action (thunk) - NHỜ middleware
store.dispatch(createAsyncThunk('todos/fetch', async () => {
  const response = await fetch('/api/todos');
  return response.json();
}));
```

## Redux-thunk Middleware hoạt động như thế nào?

```typescript
// Giả lập redux-thunk middleware
function thunkMiddleware({ dispatch, getState }) {
  return (next) => (action) => {
    // Nếu action là function → dispatch function đó
    if (typeof action === 'function') {
      return action(dispatch, getState);  // ← Truyền dispatch vào function
    }
    
    // Nếu action là object → dispatch bình thường
    return next(action);
  };
}
```

**Flow:**

```
1. Component dispatch một action
   ↓
dispatch(myThunk(payload))
   ↓
2. Redux nhận action
   ↓
3. Redux-thunk middleware kiểm tra
   ↓
typeof myThunk === 'function' ? TRUE
   ↓
4. Middleware gọi function với dispatch
   ↓
myThunk(payload, dispatch, getState)
   ↓
5. Trong function có thể gọi dispatch nhiều lần
   ↓
dispatch({ type: 'PENDING' })
dispatch({ type: 'FULFILLED', payload: data })
dispatch({ type: 'ANOTHER_ACTION' })
```

## Ví dụ chi tiết:

### Ví dụ 1: Thunk dispatch nhiều actions (thủ công)

```typescript
import { createAsyncThunk } from '@reduxjs/toolkit';

// Tạo async thunk
const fetchUsersAndPosts = createAsyncThunk(
  'users/fetchWithPosts',
  async (userId, { dispatch }) => {
    // ← Nhận dispatch từ tham số thứ 2
    
    // Dispatch action 1: Bắt đầu fetch user
    dispatch({ type: 'users/fetchPending' });
    
    try {
      // Fetch user
      const userResponse = await fetch(`/api/users/${userId}`);
      const user = await userResponse.json();
      
      // Dispatch action 2: User fetched thành công
      dispatch({ type: 'users/fetchFulfilled', payload: user });
      
      // Dispatch action 3: Bắt đầu fetch posts
      dispatch({ type: 'posts/fetchPending' });
      
      // Fetch posts của user
      const postsResponse = await fetch(`/api/users/${userId}/posts`);
      const posts = await postsResponse.json();
      
      // Dispatch action 4: Posts fetched thành công
      dispatch({ type: 'posts/fetchFulfilled', payload: posts });
      
      return { user, posts };  // ← Dispatch fulfilled action
      
    } catch (error) {
      // Dispatch action 5: Error
      dispatch({ type: 'users/fetchRejected', error: error.message });
      dispatch({ type: 'posts/fetchRejected', error: error.message });
      
      return rejectWithValue(error.message);  // ← Dispatch rejected action
    }
  }
);

// Trong component
dispatch(fetchUsersAndPosts(123));

// Flow actions được dispatch:
// 1. { type: 'users/fetchPending' }
// 2. { type: 'users/fetchFulfilled', payload: {...} }
// 3. { type: 'posts/fetchPending' }
// 4. { type: 'posts/fetchFulfilled', payload: [...] }
// 5. { type: 'users/fetchWithPosts/pending' }  ← Tự động dispatch
// 6. { type: 'users/fetchWithPosts/fulfilled', payload: {...} }  ← Tự động dispatch
```

### Ví dụ 2: `createAsyncThunk` tự động dispatch 3 actions

```typescript
const fetchUsers = createAsyncThunk('users/fetch', async (userId) => {
  const response = await fetch(`/api/users/${userId}`);
  return response.json();
});

// Khi dispatch:
dispatch(fetchUsers(123));

// Tự động dispatch 3 actions (KHÔNG cần viết tay):
// 1. { type: 'users/fetch/pending', meta: { requestId: '...', arg: 123 } }
// 2. { type: 'users/fetch/fulfilled', payload: {...} }
// 3. { type: 'users/fetch/rejected', payload: {...} } (nếu lỗi)
```

### Ví dụ 3: Trong project của bạn

```typescript
// src/assets/utils/redux.ts
const thunk = createAsyncThunk<ResponseType, PayloadType>(
  `${namespace}/REQUESTS/${name}`,
  async (payload, { dispatch, rejectWithValue }) => {
    // ← Nhận dispatch từ tham số thứ 2
    
    try {
      const response = await axiosRequest(payload.data);
      return response;
    } catch (err: any) {
      // Dispatch action global error (từ file khác)
      const globalErrorAction = getActionBasedOnErrorFromBackend(err);
      if (globalErrorAction) {
        dispatch(globalErrorAction(err));  // ← Dispatch action khác!
      }
      return rejectWithValue(err);
    }
  }
);

// Khi dispatch thunk:
dispatch(thunk({ data: payload, autoRefresh: false }));

// Actions được dispatch:
// 1. { type: '.../pending' }  ← Tự động dispatch
// 2. { type: 'globalError', ... }  ← Dispatch trong catch (nếu có lỗi)
// 3. { type: '.../rejected', payload: err }  ← Tự động dispatch
// HOẶC { type: '.../fulfilled', payload: response }  ← Tự động dispatch
```

## So sánh:

### Normal action:

```typescript
const addTodoAction = { type: 'ADD_TODO', payload: { text: 'Buy milk' } };

dispatch(addTodoAction);
// → Dispatch 1 action duy nhất
```

### Thunk action:

```typescript
const addTodoThunk = createAsyncThunk('todos/add', async (payload) => {
  // Dispatch action 1
  dispatch({ type: 'todos/pending' });
  
  try {
    const response = await fetch('/api/todos', { method: 'POST', body: payload });
    const todo = await response.json();
    
    // Dispatch action 2
    dispatch({ type: 'todos/success', payload: todo });
    
    return todo;
  } catch (error) {
    // Dispatch action 3
    dispatch({ type: 'todos/error', payload: error.message });
    throw error;
  }
});

dispatch(addTodoThunk({ text: 'Buy milk' }));

// → Dispatch 4 actions:
// 1. { type: 'todos/add/pending' }  ← Tự động
// 2. { type: 'todos/pending' }  ← Thủ công
// 3. { type: 'todos/success', ... }  ← Thủ công
// 4. { type: 'todos/add/fulfilled', ... }  ← Tự động
```

## Tóm tắt:

| Loại function | Có thể dispatch? | Khi nào có thể? |
|--------------|------------------|-----------------|
| **Normal function** | ❌ KHÔNG | Không bao giờ |
| **Action creator** | ❌ KHÔNG | Chỉ trả về object action, không dispatch |
| **Thunk (với redux-thunk)** | ✅ CÓ | Nhận dispatch từ tham số thứ 2 |

**Quy tắc:**
- Redux chỉ nhận **plain object actions**
- Để dispatch function → Cần **redux-thunk middleware**
- Thunk nhận **2 tham số**: `(payload, { dispatch, getState })`
- Trong thunk có thể gọi **dispatch nhiều lần**

### `rejectWithValue` được thiết kế để:

1. __Cho phép custom error payload__ - chứa nhiều thông tin hơn (message, code, timestamp, ...)
2. __Tách biệt với `action.error`__ - `action.error` là error object mặc định, `action.payload` là error custom


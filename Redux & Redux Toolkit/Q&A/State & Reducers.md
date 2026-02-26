# Trong Redux, state là immutable có nghĩa là gì và tại sao cần immutable state?

**State immutability** trong Redux có nghĩa là khi update state, KHÔNG ĐƯỢC thay đổi state hiện tại mà thay vào đó tạo ra một state mới. Điều này đảm bảo tính toàn vẹn của state và giúp dễ dàng hơn trong việc debug và trace lại thay đổi.

### Tại sao cần Immutable State?

Có __4 lý do chính__:

---

## 1. Predictability (Dễ dự đoán)

__Vấn đề với mutable state:__

```typescript
// ❌ SAi: Mutable state
let state = { count: 0, items: [] };

function addItem(state, item) {
  state.items.push(item);  // ← Mutate state gốc
  return state;
}

const result = addItem(state, { id: 1 });
console.log(state);  // ← State gốc đã bị thay đổi!
console.log(result);  // ← Result trỏ đến state gốc
console.log(state === result);  // ← true (cùng object)
```

__Kết quả:__

- State gốc bị thay đổi → __Side effect__ không mong muốn
- Khó debug vì không biết ai đã sửa state
- Khó test vì phụ thuộc vào thứ tự thực hiện

## 2. Change Detection (Phát hiện thay đổi)

__Vấn đề với mutable state:__

```typescript
// ❌ SAi: Khó phát hiện thay đổi
let state = { items: ['A', 'B'] };
const previousState = state;

state.items.push('C');  // ← Mutate

console.log(previousState === state);  // ← true (vẫn trỏ cùng object)
console.log(previousState.items);  // ← ['A', 'B', 'C'] (đã bị thay đổi!)

// React không biết state đã thay đổi
// → Component KHÔNG re-render
```
## 3. Performance Optimization (Tối ưu hiệu năng)

__React chỉ re-render khi prop/state thay đổi:__

```typescript
// ✅ Đúng: Immutable state giúp React tối ưu
function MyComponent({ todos }) {
  // React chỉ re-render khi todos === previousTodos là false
  // → Khi todos là object mới (immutable)
  // → KHÔNG re-render khi todos vẫn là object cũ (mutable)
}

// Nếu mutate state:
const todos = [{ id: 1, text: 'Todo 1' }];
todos.push({ id: 2, text: 'Todo 2' });  // ← Mutate

// React KHÔNG biết todos đã thay đổi (reference vẫn như cũ)
// → Component KHÔNG re-render (BUG!)

// Nếu dùng immutable:
const todos = [{ id: 1, text: 'Todo 1' }];
const newTodos = [...todos, { id: 2, text: 'Todo 2' }];  // ← Tạo array mới

// React BIẾT todos đã thay đổi (reference khác)
// → Component re-render (ĐÚNG!)
```

---

## 4. Time Travel Debugging (Debug với Redux DevTools)

__Redux DevTools có thể undo/redo vì immutable:__

```typescript
// Redux lưu lại lịch sử các state

Action 1: ADD_TODO
State:     { todos: [{ id: 1, text: 'Todo 1' }] }

Action 2: ADD_TODO
State:     { todos: [{ id: 1, text: 'Todo 1' }, { id: 2, text: 'Todo 2' }] }

Action 3: DELETE_TODO
State:     { todos: [{ id: 1, text: 'Todo 1' }] }

// ✓ Có thể undo về Action 1
// ✓ Có thể redo đến Action 3
// ✓ Vì mỗi state là object mới, không bị mutate
```

__Nếu mutate state:__

```typescript
// ❌ SAi: Mutable state

Action 1: ADD_TODO
State:     { todos: [{ id: 1, text: 'Todo 1' }] }

Action 2: ADD_TODO
State:     { todos: [{ id: 1, text: 'Todo 1' }, { id: 2, text: 'Todo 2' }] }
           // ← Nhưng thực tế là cùng object với Action 1!

// ✗ KHÔNG thể undo/redo
// ✗ Không biết state trước đó là gì
```

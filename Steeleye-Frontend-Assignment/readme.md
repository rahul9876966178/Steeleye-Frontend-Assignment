# Explain what the simple List component does.
- `List` is memoised list `WrappedListComponent` which takes only one prop `items` that is an array of object where each object must have a `text` property with the type of its value as string - the value of `items` by default is set to **null**
- It has a state variable `selectedState` to store the index of item that was selected
- Every time the prop `items` changes (or during the initial render) `selectedIndex` is being set to **null**; hence, no item is selected anymore (if any).
- It also has a function `handleClick` which takes a parameter `index`, the function upon being ca  lled sets the `selectedIndex` to `index` by calling `setSelectedIndex` function.
- `items` is being traversed over and each item is rendered as `SingleListItem`
- `SingleListItem` is memoised list item `WrappedSingleListItem` whose background is _RED_ by default but turns _GREEN_ when selected, only one out of all the items can be selected at a time.
- It has four parameters, two out of them is required which is `text` and `onClickHandler` and other two `index` and `isSelected` are optional.

# What problems / warnings are there with code?

1. PropTypes.shapeOf is not a function 
2. Calling PropTypes validators directly is not supported by the `prop-types` package. Basically, `PropTypes.array` is a validator and it is not callable.
3. Cannot read properties of null. By default, the value of `items` is null and we are traversing over it without checking if it is null or not
4. `setSelectedIndex` is not a function. The order of destructuring useState in `WrappedListComponent` is wrong.
5. Each child in a list should have a unique **key** prop.
6. The value we are expecting for `isSelected` prop of `WrappedSingleListItem` is of type `boolean` but we are passing `selectedIndex` as its value which is of type `number` when calling the component while iterating over items in `WrappedListComponent`
7. Cannot update a component `WrappedListComponent` while rendering a different component `WrappedSingleListItem`. In `WrappedSingleListItem`, the value of `onClick` is a function call, which means when `WrappedSingleListItem` renders, it calls that function and tries to update another component, which in this case is its parent component  `WrappedListComponent`.
8. Although we have used `memo` to prevent `WrappedSingleListItem` from re-rendering itself in case there is no change in props, it still re-renders becuse we are creating a new instance of the function `() => handleClick(index)` everytime the `WrappedListComponent` renders/re-renders.
9. `isSelected` prop is not required. It tells which list item is selected and which are not, in case the developer forgets to pass it, the list item will always be marked unselected.

# Please fix, optimize, and/or modify the component as much as you think is necessary.

## Fixes
### 1.
```diff
-PropTypes.shapeOf
+PropTypes.shape
``` 
### 2.
```diff
-PropTypes.array
+PropTypes.arrayOf
```

After fixing these two the relevant piece of code is:
```js
WrappedListComponent.propTypes = {
  items: PropTypes.arrayOf(
    PropTypes.shape({
      text: PropTypes.string.isRequired,
    })
  ),
};
```

### 3. We should check if the `items` is **null** or not before traversing over it. We can do so by using different methods like using if-else, conditional operator or optional chaining. I've fixed the issue using optional chaining as it requires the least amount of changes to be made and clears the intent just as same as any other method.
```js
{items?.map((item, index) => (
  <SingleListItem
    onClickHandler={() => handleClick(index)}
    text={item.text}
    index={index}
    isSelected={selectedIndex}
  />
))}
```

### 4. The order in which we are destructuring useState is wrong. We are first expecting a function and then a value: which represents the current state, but the order should be value and then the function.

```diff
-const [setSelectedIndex, selectedIndex] = useState();
+const [selectedIndex, setSelectedIndex] = useState();
```

### Although it is optional in this case, we should pass a default value to `useState` which represents the initial state.

```js
const [selectedIndex, setSelectedIndex] = useState(null);
```

### 5. Keys help React identify which items have changed, are added, or are removed. Keys should be given to the elements inside the array to give the elements a stable identity. In case if we don't have an ID that can give elements inside array a stable identity, we may use the index as key but doing so is not recommended if the order of items changes, item is removed, and/or inserted. 
> In this case, I'm assuming the there won't be any changes in the `items` since the code has no function for doing so. Thus, I've used the index of the item itself as the key.
```js
<SingleListItem
  key={index}
  onClickHandler={() => handleClick(index)}
  text={item.text}
  index={index}
  isSelected={selectedIndex}
/>
```

### 6. Since we are expecting a boolean value for `isSelected` which simply represents if current index is the selected index, we should just compare the `selectedIndex` with `index` for equality and pass the result to `isSelected`

```js
<SingleListItem
  key={index}
  onClickHandler={() => handleClick(index)}
  text={item.text}
  index={index}
  isSelected={selectedIndex === index}
/>
```

### 7. 
```diff
const WrappedSingleListItem = ({ index, isSelected, onClickHandler, text }) => {
  return (
    <li
      style={{ backgroundColor: isSelected ? "green" : "red" }}
-     onClick={onClickHandler(index)}
+     onClick={() => onClickHandler(index)}
    >
      {text}
    </li>
  );
};
```

### 8. We need to memoise the `() => handleClick(index)` function which is passed as value to the `onClickHandler` prop of `SingleListItem` component. So that we use the same instance of function for same index or only one instance of function for every index. So can be achieved by using the `useCallback` hook.
```js
const handleClick = useCallback((index) => {
  setSelectedIndex(index);
}, []);

// it has an empty array as dependency means the same function will be re-used, for every subsequent re-renders of [WrappedListComponent], that was created during the initial render of the component.
```
```diff
<ul style={{ textAlign: "left" }}>
  {items?.map((item, index) => (
    <SingleListItem
      key={index}
-     onClickHandler={() => handleClick(index)}
+     onClickHandler={handleClick}
      text={item.text}
      index={index}
      isSelected={selectedIndex == index}
    />
  ))}
</ul>
```
> Since `handleClick` will now need an index parameter be passed to it from the component `SingleListItem` to mark it selected, we must now necessarily pass the `index` prop. Hence we should mark it required.

```js
WrappedSingleListItem.propTypes = {
  index: PropTypes.number.isRequired,
  isSelected: PropTypes.bool,
  onClickHandler: PropTypes.func.isRequired,
  text: PropTypes.string.isRequired,
};
```

### 9. Mark the `isSelected` prop required.
```js
WrappedSingleListItem.propTypes = {
  index: PropTypes.number.isRequired,
  isSelected: PropTypes.bool.isRequired,
  onClickHandler: PropTypes.func.isRequired,
  text: PropTypes.string.isRequired,
};
```

# Changes I've made

1. When the items list is empty, although we won't have any list item displayed, we'll have an empty `ul` tag in the DOM Tree, we should conditionally render the list only if it is not null and not empty.
```js
if (items == null || items.length == 0) {
  return null;
  // or, we can return something else that shows the list is empty
}

return (
  <ul style={{ textAlign: "left" }}>
    {items.map((item, index) => (
      <SingleListItem
        key={index}
        onClickHandler={handleClick}
        text={item.text}
        index={index}
        isSelected={selectedIndex == index}
      />
    ))}
  </ul>
);
```

2. Each item in `items` should have an id that makes it identity unique and stable in the `items` array. Although it is unnecessary to do it in this case but in future, we may implement the feature of insertion, deletion, or re-ordering the items.
```js
WrappedListComponent.propTypes = {
  items: PropTypes.arrayOf(
    PropTypes.shape({
      text: PropTypes.string.isRequired,
      id: PropTypes.string.isRequired
    })
  ),
};
```


* We should also consider keeping each component in a different file
* In case the no of items is too large, we should considering using `react-window` or something equivalent. `react-window` works by only rendering part of a large data set (just enough to fill the viewport). This helps address some common performance bottlenecks:
  1. It reduces the amount of work (and time) required to render the initial view and to process updates.
  2. It reduces the memory footprint by avoiding over-allocation of DOM nodes.
---
## The updated code is:
```js
import React, { useState, useEffect, useCallback, memo } from "react";
import PropTypes from "prop-types";

const WrappedSingleListItem = ({ index, isSelected, onClickHandler, text }) => {
  return (
    <li
      style={{ backgroundColor: isSelected ? "green" : "red" }}
      onClick={() => onClickHandler(index)}
    >
      {text}
    </li>
  );
};

WrappedSingleListItem.propTypes = {
  index: PropTypes.number.isRequired,
  isSelected: PropTypes.bool.isRequired,
  onClickHandler: PropTypes.func.isRequired,
  text: PropTypes.string.isRequired,
};

const SingleListItem = memo(WrappedSingleListItem);

const WrappedListComponent = ({ items }) => {
  const [selectedIndex, setSelectedIndex] = useState(null);

  useEffect(() => {
    setSelectedIndex(null);
  }, [items]);

  const handleClick = useCallback((index) => {
    setSelectedIndex(index);
  }, []);

  if (items == null || items.length == 0) {
    return null;
  }

  return (
    <ul style={{ textAlign: "left" }}>
      {items.map((item, index) => (
        <SingleListItem
          key={item.id}
          onClickHandler={handleClick}
          text={item.text}
          index={index}
          isSelected={selectedIndex == index}
        />
      ))}
    </ul>
  );
};

WrappedListComponent.propTypes = {
  items: PropTypes.arrayOf(
    PropTypes.shape({
      text: PropTypes.string.isRequired,
      id: PropTypes.string.isRequired
    })
  ),
};

WrappedListComponent.defaultProps = {
  items: null,
};

const List = memo(WrappedListComponent);

export default List;
```

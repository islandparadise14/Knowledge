---
tags: [android, recyclerview, adapter]
---

# RecyclerView 기초

## 💡 핵심 개념

**RecyclerView**는 대량의 데이터를 효율적으로 표시하는 View다. ViewHolder 패턴으로 View를 재사용한다.

## 🔍 자세히

### 기본 구성

```kotlin
// Adapter
class MyAdapter(private val items: List<Item>) : 
    RecyclerView.Adapter<MyAdapter.ViewHolder>() {
    
    class ViewHolder(val binding: ItemBinding) : 
        RecyclerView.ViewHolder(binding.root)
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ViewHolder(binding)
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.binding.textView.text = items[position].name
    }
    
    override fun getItemCount() = items.size
}

// Activity
recyclerView.layoutManager = LinearLayoutManager(this)
recyclerView.adapter = MyAdapter(items)
```

### LayoutManager

```kotlin
// Vertical List
LinearLayoutManager(context)

// Horizontal List
LinearLayoutManager(context, LinearLayoutManager.HORIZONTAL, false)

// Grid
GridLayoutManager(context, spanCount)

// Staggered Grid
StaggeredGridLayoutManager(spanCount, orientation)
```

### DiffUtil

```kotlin
class MyDiffCallback(
    private val oldList: List<Item>,
    private val newList: List<Item>
) : DiffUtil.Callback() {
    
    override fun getOldListSize() = oldList.size
    override fun getNewListSize() = newList.size
    
    override fun areItemsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos].id == newList[newPos].id
    }
    
    override fun areContentsTheSame(oldPos: Int, newPos: Int): Boolean {
        return oldList[oldPos] == newList[newPos]
    }
}

// 사용
val diffResult = DiffUtil.calculateDiff(MyDiffCallback(oldList, newList))
diffResult.dispatchUpdatesTo(adapter)
```

### ListAdapter

```kotlin
class MyAdapter : ListAdapter<Item, MyAdapter.ViewHolder>(DiffCallback()) {
    
    class DiffCallback : DiffUtil.ItemCallback<Item>() {
        override fun areItemsTheSame(oldItem: Item, newItem: Item) = 
            oldItem.id == newItem.id
        
        override fun areContentsTheSame(oldItem: Item, newItem: Item) = 
            oldItem == newItem
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        // ...
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val item = getItem(position)  // ListAdapter 메서드
        // ...
    }
}

// 사용
adapter.submitList(newList)  // DiffUtil 자동 처리
```

### 최적화

**1. setHasFixedSize**
```kotlin
recyclerView.setHasFixedSize(true)  // 크기 고정 시
```

**2. RecyclerView Pool**
```kotlin
val sharedPool = RecyclerView.RecycledViewPool()
recyclerView1.setRecycledViewPool(sharedPool)
recyclerView2.setRecycledViewPool(sharedPool)
```

**3. Prefetch**
```kotlin
// 자동으로 처리됨 (LinearLayoutManager)
```

## 💻 실전 예제

### Click Listener
```kotlin
class MyAdapter(
    private val onItemClick: (Item) -> Unit
) : ListAdapter<Item, MyAdapter.ViewHolder>(DiffCallback()) {
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        val item = getItem(position)
        holder.itemView.setOnClickListener {
            onItemClick(item)
        }
    }
}

// 사용
val adapter = MyAdapter { item ->
    // 클릭 처리
}
```

## 🔗 관련 개념

- [[04-View-System/RecyclerView/DiffUtil]]
- [[05-Jetpack-Compose/리스트/LazyColumn]]

---

**핵심 요약:** ViewHolder로 재사용. ListAdapter + DiffUtil 권장. setHasFixedSize로 최적화.

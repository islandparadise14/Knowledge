---
tags: [android, lifecycle, fragment]
---

# Fragment Lifecycle

## 💡 핵심 개념

Fragment는 Activity보다 복잡한 생명주기를 가진다. **onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume → onPause → onStop → onDestroyView → onDestroy → onDetach**

## 📌 왜 필요한가?

Fragment는 재사용 가능하고 Activity보다 유연하다. 하지만 생명주기가 복잡해서 제대로 이해하지 못하면 메모리 누수, 크래시가 발생한다.

## 🔍 자세히

### 생명주기 흐름

```
onAttach() → onCreate() → onCreateView() → onViewCreated()
    ↓
onStart() → onResume() [RESUMED]
    ↓
onPause() → onStop() → onDestroyView()
    ↓
onDestroy() → onDetach()
```

### 주요 콜백

```kotlin
class MyFragment : Fragment() {
    // 1. Activity에 붙음
    override fun onAttach(context: Context) {
        super.onAttach(context)
    }
    
    // 2. Fragment 생성
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ViewModel 생성, arguments 읽기
    }
    
    // 3. View 생성
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return inflater.inflate(R.layout.fragment_my, container, false)
    }
    
    // 4. View 생성 완료
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        // View 초기화, 리스너 설정
        binding.button.setOnClickListener { }
    }
    
    // 5. View 파괴
    override fun onDestroyView() {
        // binding 정리, 리스너 해제
        _binding = null
        super.onDestroyView()
    }
}
```

### View Binding 패턴

```kotlin
class MyFragment : Fragment() {
    private var _binding: FragmentMyBinding? = null
    private val binding get() = _binding!!
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentMyBinding.inflate(inflater, container, false)
        return binding.root
    }
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.textView.text = "Hello"
    }
    
    override fun onDestroyView() {
        _binding = null  // 필수!
        super.onDestroyView()
    }
}
```

### Fragment vs Activity 생명주기

```kotlin
Activity: onCreate → onStart → onResume
Fragment: onAttach → onCreate → onCreateView → onViewCreated → onStart → onResume

Activity: onPause → onStop → onDestroy
Fragment: onPause → onStop → onDestroyView → onDestroy → onDetach
```

### ViewLifecycleOwner

```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // viewLifecycleOwner 사용 (권장)
        viewModel.data.observe(viewLifecycleOwner) { data ->
            // View가 있을 때만 호출
        }
        
        // this 사용 금지
        // viewModel.data.observe(this) { }  // 메모리 누수!
    }
}
```

### 주의사항

**1. Binding 정리 필수**

```kotlin
// 나쁨 - 메모리 누수
private val binding: FragmentMyBinding? = null

override fun onCreateView(...): View {
    binding = FragmentMyBinding.inflate(...)
    return binding.root
}
// onDestroyView에서 정리 안 함 → 누수!

// 좋음
private var _binding: FragmentMyBinding? = null
override fun onDestroyView() {
    _binding = null
    super.onDestroyView()
}
```

**2. ViewLifecycleOwner 사용**

```kotlin
// 나쁨
viewModel.data.observe(this) { }

// 좋음
viewModel.data.observe(viewLifecycleOwner) { }
```

**3. viewLifecycleScope 사용**

```kotlin
// onViewCreated에서
viewLifecycleOwner.lifecycleScope.launch {
    viewModel.state.collect { }
}
```

## 💻 실전 예제

### Compose 통합

```kotlin
class MyFragment : Fragment() {
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return ComposeView(requireContext()).apply {
            setContent {
                MyScreen()
            }
        }
    }
}
```

### Fragment Result API

```kotlin
// 데이터 전달
setFragmentResult("requestKey", bundleOf("key" to "value"))

// 데이터 수신
setFragmentResultListener("requestKey") { _, bundle ->
    val result = bundle.getString("key")
}
```

## 🔗 관련 개념

- [[03-Android-Framework/생명주기/Activity-Lifecycle]]
- [[03-Android-Framework/Fragment/Fragment-Manager]]

## 📚 더 보기

- [Android Developers - Fragment Lifecycle](https://developer.android.com/guide/fragments/lifecycle)

---

**핵심 요약:** onCreateView에서 View 생성, onViewCreated에서 초기화, onDestroyView에서 정리. viewLifecycleOwner 필수.

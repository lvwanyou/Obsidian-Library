| **生命周期阶段** | **Activity 方法**          | **Fragment 方法**          | **关键区别**                                                  |
| ---------- | ------------------------ | ------------------------ | --------------------------------------------------------- |
| **创建**     | `onCreate()`             | `onCreate()`             | Fragment 的 `onCreate()` 在 Activity 的 `onCreate()` 之后调用。   |
| **视图创建**   | `onCreateView()` ❌       | `onCreateView()`         | Fragment 需要在此返回布局视图，Activity 通过 `setContentView()` 设置布局。  |
| **可见性变化**  | `onStart()`/`onStop()`   | `onStart()`/`onStop()`   | Fragment 的可见性受 Activity 和自身状态双重影响（如 `add()`/`detach()`）。  |
| **用户交互状态** | `onResume()`/`onPause()` | `onResume()`/`onPause()` | Fragment 的 `onResume()` 在 Activity 的 `onResume()` 之后调用。   |
| **销毁视图**   | `onDestroyView()` ❌      | `onDestroyView()`        | Fragment 视图被移除时调用，但实例可能仍存在（如通过 `addToBackStack()`）。       |
| **完全销毁**   | `onDestroy()`            | `onDestroy()`            | Fragment 的 `onDestroy()` 在 Activity 的 `onDestroy()` 之前调用。 |

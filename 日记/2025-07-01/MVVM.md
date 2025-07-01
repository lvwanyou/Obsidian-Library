## **2. MVVM 的工作流程**

1. **用户操作 View**（如点击按钮）→ **触发 ViewModel 命令**。
    
2. **ViewModel 更新 Model**（如调用 API 修改数据）。
    
3. **Model 数据变化** → **自动通知 ViewModel**。
    
4. **ViewModel 更新数据** → **View 自动刷新**（通过数据绑定）78。
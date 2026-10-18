---
title: 【C# 實作 05】StudentManagement：WPF 的 MVVM 搭配 MVP
date: 2026-10-18 20:50:00
updated: 2026-10-18 20:50:00
categories:
  - C#
  - 桌面應用
tags:
  - C#
  - WPF
  - 設計模式
---

這是我第一個 WPF 專案：一個可以新增、編輯、刪除學生資料的小工具。重點不在功能，而是練習 XAML、Binding、Command，以及把 MVVM 和之前學過的 MVP 組合在一起。

<!-- more -->

# 專案簡介

StudentManagement 是一個單視窗的學生資料管理程式：上方表單輸入姓名、年紀、身高、體重後新增，下方以卡片列表顯示每位學生，每張卡片可以勾選、編輯、存檔與刪除（必須先勾選才能刪除）。

- 平台：**.NET Framework 4.8**、WPF
- NuGet 套件：`Fody` 6.9.3、`PropertyChanged.Fody` 4.1.0（`FodyWeavers.xml` 啟用 `<PropertyChanged />`）
- Repo：[https://github.com/XiaoXiaTian2/StudentManagement](https://github.com/XiaoXiaTian2/StudentManagement)

# 架構與功能

```text text
StudentManagement/
├── MainWindow.xaml / .xaml.cs      View：XAML 版面，DataContext 指向 MainViewModel
├── MainViewModel.cs                ViewModel，同時實作 MVP 的 IStudentView
├── Presenters/StudentPresenter.cs  Presenter：持有 StudentDTO 清單，處理增刪
├── Contracts/StudentContract.cs    IStudentPresenter / IStudentView 介面
├── DTOs/StudentDTO.cs              Presenter 端使用的資料物件
├── Student.cs                      畫面上的學生物件，含 IsEditable 等 UI 狀態
├── RelayCommand.cs                 ICommand 實作（含泛型版）
└── Converters/InverseBooleanToVisibilityConverter.cs  bool 轉 Visibility
```

| 角色 | 類別 | 負責 |
| --- | --- | --- |
| View | `MainWindow.xaml` | 版面、Style、Binding，幾乎沒有 code-behind |
| ViewModel | `MainViewModel` | 提供 Binding 用的屬性與 Command，接收 Presenter 的回呼 |
| Presenter | `StudentPresenter` | 轉型、維護資料清單 |
| Model | `StudentDTO`、`Student` | DTO 給 Presenter，`Student` 給畫面 |

code-behind 只做一件事：

```csharp csharp
public MainWindow()
{
    InitializeComponent();
    DataContext = new MainViewModel();
}
```

# 核心知識點

## 1. MVVM 搭配 MVP：ViewModel 就是 Presenter 眼中的 View

在 WinForms 專案裡我習慣用 MVP 的 Contract 寫法：把 Presenter 與 View 的介面放在同一個 Contract 類別裡，彼此只認識介面。

```csharp csharp
internal class StudentContract
{
    public interface IStudentPresenter
    {
        void AddStudent(string name, string age, string height, string weight);
        void DeleteStudent(Student student);
        void EditStudent(Student student);
        void SaveStudent(Student student);
    }

    public interface IStudentView
    {
        // 取得資料的Callback
        void OnAddStudent(List<StudentDTO> dtos);
        void OnDeleteStudent(List<StudentDTO> dtos);
        void OnEditStudent(Student std);
        void OnSaveStudent(Student std);
    }
}
```

到了 WPF，畫面改由 Binding 驅動，所以我讓 `MainViewModel` 去實作 `IStudentView`。對 Presenter 來說它就是一個 View；對 XAML 來說它是 ViewModel。Command 收到操作後轉交給 Presenter，Presenter 處理完再呼叫 `OnXxx` 回呼更新 ViewModel 的屬性：

```csharp csharp
[AddINotifyPropertyChangedInterface]
internal class MainViewModel : IStudentView
{
    // ...
    public MainViewModel()
    {
        IStudentPresenter presenter = new StudentPresenter(this);
        this.AddStudentCommand = new RelayCommand(() =>
        {
            presenter.AddStudent(Name, Age, Height, Weight);
        });
        this.DeleteCommand = new RelayCommand<Student>(presenter.DeleteStudent);
        this.EditCommand = new RelayCommand<Student>(presenter.EditStudent);
        this.SaveCommand = new RelayCommand<Student>(presenter.SaveStudent);
    }
    // ...
}
```

這樣做的好處是：字串轉數字、維護清單這類邏輯集中在 Presenter，ViewModel 只負責「把資料變成畫面能綁的形狀」。Presenter 端用 `StudentDTO`，畫面端用帶 UI 狀態（`IsEditable`、`IsSavable`）的 `Student`，兩邊的關注點也分開了。

## 2. ICommand 與 `RelayCommand<T>`

WPF 的 Button 透過 `Command` 屬性綁定 `ICommand`，取代 WinForms 的 Click 事件。我寫了一般版與泛型版的 `RelayCommand`，泛型版把 `CommandParameter` 轉成指定型別再交給委派：

```csharp csharp
internal class RelayCommand<T> : ICommand
{
    public event EventHandler CanExecuteChanged;

    Action<T> action;
    public RelayCommand(Action<T> action)
    {
        this.action = action;
    }

    public bool CanExecute(object parameter)
    {
        return true;
    }

    public void Execute(object parameter)
    {
        action.Invoke((T)parameter);
    }
}
```

列表中的每張卡片的 DataContext 是 `Student`，但 Command 定義在 `MainViewModel`。所以按鈕要用 `RelativeSource AncestorType=Window` 往上找到 Window 的 DataContext，再用 `CommandParameter="{Binding}"` 把目前這筆 `Student` 傳進去：

```xml xml
<Button
    Margin="8,0,0,0"
    Command="{Binding DataContext.DeleteCommand, RelativeSource={RelativeSource AncestorType=Window}}"
    CommandParameter="{Binding}"
    Content="✕ 刪除"
    DockPanel.Dock="Right"
    IsEnabled="{Binding IsDeletable}"
    Style="{StaticResource DangerButton}"
    Tag="{Binding Name}" />
```

這是我在這個專案卡最久的地方：在 `DataTemplate` 裡直接寫 `{Binding DeleteCommand}` 會去 `Student` 上找，當然找不到。

## 3. INotifyPropertyChanged：手寫 vs. Fody

Binding 要在資料變動時更新畫面，物件必須實作 `INotifyPropertyChanged`。`Student` 是手寫版本，可以看到一個屬性變動時同時通知另一個衍生屬性（勾選 `IsCompleted` 後 `IsDeletable` 也要重新讀取）：

```csharp csharp
private bool _isCompleted;

public bool IsCompleted
{
    get => _isCompleted;
    set
    {
        if (_isCompleted != value)
        {
            _isCompleted = value;
            OnPropertyChange(nameof(IsCompleted));
            OnPropertyChange(nameof(IsDeletable));
        }
    }
}

public bool IsDeletable
{
    get => _isCompleted;
}
// ...
private void OnPropertyChange(string propertyName)
{
    this.PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
}
```

`MainViewModel` 則改用 Fody：只要加上 `[AddINotifyPropertyChangedInterface]`，編譯時就會自動織入通知程式碼，auto-property 寫法即可。原本手寫的 `OnPropertyChange` 還留在註解裡，可以對照兩種寫法：

```csharp csharp
[AddINotifyPropertyChangedInterface]
internal class MainViewModel : IStudentView
{
    //private string _name;

    public string Name { get; set; }
    //private string _age;
    public string Age { get; set; }
    public string Height { get; set; }
    public string Weight { get; set; }

    public ObservableCollection<Student> students { get; set; }
    // ...
}
```

`OnAddStudent` 每次都會整個換掉 `students`，因為 Fody 會替 `students` 屬性發出通知，`ItemsControl` 才會重新綁定新的集合。

## 4. IValueConverter：同一個欄位的顯示／編輯切換

每個欄位同時放了「顯示用的 TextBlock」和「編輯用的 TextBox」，用 `IsEditable` 決定誰看得到。轉換器加上 `ConverterParameter` 讓同一個類別可以正反兩用：

```csharp csharp
public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
{
    if (value is bool b)
    {
        if (parameter?.ToString() == "inverse")
        {
            b = !b;
        }
        return b ? Visibility.Collapsed : Visibility.Visible;
    }
    return Visibility.Visible;
}
```

```xml xml
<Grid Style="{StaticResource FieldCell}">
    <StackPanel Orientation="Horizontal" Visibility="{Binding IsEditable, Converter={StaticResource VisibilityConverter}, ConverterParameter=inverse}">
        <TextBlock Style="{StaticResource LabelText}" Text="姓名" />
        <TextBlock Style="{StaticResource ValueText}" Text="{Binding Name}" />
    </StackPanel>
    <TextBox
        Width="120"
        Text="{Binding Name}"
        Visibility="{Binding IsEditable, Converter={StaticResource VisibilityConverter}}" />
</Grid>
```

`IsEditable` 的意思是「編輯按鈕可以按」，也就是目前處於顯示模式：此時 TextBlock 顯示、TextBox 收起。按下編輯後 `OnEditStudent` 把 `IsEditable` 設成 false、`IsSavable` 設成 true，兩者對調。回頭看，這個命名和轉換器名稱都不太直覺，用 `IsEditing` 之類的名字會更好讀。

## 5. XAML 的 Style、ControlTemplate 與 BasedOn

外觀全部寫在 `Window.Resources`。按鈕先定義一個 `BaseButton`，用 `ControlTemplate` 換掉預設外框做出圓角，再用 `BasedOn` 衍生出不同顏色的按鈕，只覆寫背景色與滑鼠移入的 Trigger：

```xml xml
<Style x:Key="BaseButton" TargetType="Button">
    <!-- ... -->
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border
                    x:Name="border"
                    Padding="{TemplateBinding Padding}"
                    Background="{TemplateBinding Background}"
                    CornerRadius="6"
                    SnapsToDevicePixels="True">
                    <!-- ... -->
                </Border>
                <ControlTemplate.Triggers>
                    <Trigger Property="IsEnabled" Value="False">
                        <Setter TargetName="border" Property="Background" Value="{StaticResource DisabledBrush}" />
                        <!-- ... -->
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>

<Style
    x:Key="PrimaryButton"
    BasedOn="{StaticResource BaseButton}"
    TargetType="Button">
    <Setter Property="Background" Value="{StaticResource PrimaryBrush}" />
    <Style.Triggers>
        <Trigger Property="IsMouseOver" Value="True">
            <Setter Property="Background" Value="{StaticResource PrimaryHoverBrush}" />
        </Trigger>
    </Style.Triggers>
</Style>
```

`TemplateBinding` 讓 Template 內的 Border 沿用按鈕本身的 `Background`、`Padding`，所以 `PrimaryButton`、`SuccessButton`、`DangerButton` 只要改 Setter 就能共用同一個外框。配合 `IsEnabled` 的 Trigger，按鈕停用時（例如沒勾選就不能刪除）會自動變灰。

# 待改進

重讀程式碼時確認到以下問題：

1. **刪除永遠刪到第一筆**：`Presenters/StudentPresenter.cs` 的 `AddStudent` 建立 `StudentDTO` 時沒有指定 `Id`，所有 DTO 的 `Id` 都是 `Guid.Empty`；`Student` 物件也從未設定 `Id`。`DeleteStudent` 用 `this.students.First(x => x.Id == student.Id)` 尋找，條件對每一筆都成立，結果不論按哪一張卡片的刪除，都會刪掉清單中的第一筆。修正方向：`AddStudent` 時給 `Id = Guid.NewGuid()`，並在轉成 `Student` 時一併帶過去。
2. **體重被身高覆蓋、Id 沒帶回畫面**：`MainViewModel.cs` 的 `OnAddStudent` 與 `OnDeleteStudent` 都寫成 `new Student(d.Name, d.Age, d.Height, d.Height)`，第四個參數應該是 `d.Weight`，所以畫面上的體重永遠顯示成身高；同時也沒有把 `d.Id` 指定給 `Student`，即使修好第 1 點，刪除時傳回 Presenter 的 Id 仍是空的。
3. **編輯後的資料沒有存回 Presenter**：`StudentPresenter.SaveStudent` 只呼叫 `OnSaveStudent` 切換 `IsEditable` / `IsSavable`，沒有更新 Presenter 持有的 `StudentDTO` 清單。之後只要新增或刪除，`OnAddStudent` / `OnDeleteStudent` 會從 DTO 清單重建整個 `students`，先前編輯過的內容就被還原。
4. **輸入未驗證就 Parse**：`AddStudent` 直接 `int.Parse(age)`、`float.Parse(height)`、`float.Parse(weight)`。欄位空白（屬性為 null）或輸入非數字時會丟出例外，而 Command 裡沒有任何處理，程式會直接結束。可以改用 `TryParse`，失敗時透過 View 回呼顯示錯誤訊息。

# 小結

這個專案讓我從 WinForms 的事件驅動轉換到 WPF 的 Binding 思維：Command 取代 Click、Converter 取代手動切換 Visible、Style 取代逐一設定屬性。把 MVP 的 Presenter 留下來、讓 ViewModel 實作 View 介面，是我自己嘗試的組合，職責確實比較清楚；但重讀後也發現資料識別（Id）和雙向同步沒有處理好，這是之後要補的基本功。

---

- 系列目錄：{% post_link csharp/csharp-course-review 'C# 課程實作總整理' %}
- 上一篇：{% post_link csharp/desktop/youtuber-subscribe '【C# 實作 04】YoutuberSubcribe' %}
- 下一篇：{% post_link csharp/projects/accounting-app '【C# 實作 06】AccountingApp' %}

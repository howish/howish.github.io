# Python中模擬於runtime時決定父類別對象的方法

## 前言
最近碰到一個問題是在設計一套API時，想讓使用者(其實就是我自己)在run time時選擇要讓使用的Interface繼承哪一個父類別。

情境大概如下
```Python

class Pencil:
    def draw_line(self):
        print('Draw line with pencil')

class MarkPen:
    def draw_line(self):
        print('Draw line with mark pen')

class Brush:
    def __init__(self, brush_name)
        # Initialize the brush with choosed brush type
        
    def draw_line(self):
        print('You have a brush')
        <super_class>.draw_line(self) # ??
        
    def draw_2_line(self):
        self.draw_line()
        self.draw_line()
```
在以上情境中，我想做到在我建立一個Brush時可以指定要繼承哪一種brush type，並且在執行self.draw_2_line()時可以用我指定的brush畫兩次線，甚或是改寫選擇的父類別的method。

雖然這例子看起來要workaround很簡單，只要在init時存一個brush實例就好：
```Python
class Brush:
    def __init__(self, brush_name)
        self._brush = choose_brush(brush_name)()
        
    def draw_2_line(self)
        self._brush.draw_line()
        self._brush.draw_line()
```
不過一來實際情形中Brush的擴充可能很大，若每一個method都要拿_brush出來call，既不直覺也不好管理。
二來是在設計架構的概念上Brush class的層級是與Pencil或MarkPen同一層的，是它的一種擴充而不是wrapper，這樣寫不太符合設計的概念。


## 最初的Workaround

在我寫出上面的template之前，我是這樣work around的:
```Python
class Pencil:
    def draw_line():
        print('Draw line with pencil')

class MarkPen:
    def draw_line():
        print('Draw line with mark pen')

def choose_brush(brush_name):
    if brush_name == 'pencil'
        return Pencil
    else:
        return MarkPen   

def get_drawer(brush_name):
    _super_class = choose_brush(brush_name)
    class Brush(_super_class):
        def __init__(self)
            pass

        def draw_line(self):
            print('You have a brush.')
            _super_class.draw_line(self) # Just do common method inheriting.
        
        def draw_2_lines(self)
            self.draw_line()
            self.draw_line()
    return Brush

# Usage
drawer = get_drawer('pencil')()
drawer.draw_line()
drawer.draw_2_lines()
```
雖然這寫法的效果完全照我的想法走。不過這種寫法強行將class包在一個function裡面，當Brush這個class越來越大時，我就越看越不順眼，決定思考一些較漂亮的寫法。

## 更多workaround

### 用 magic method 模擬attribute繼承
改寫 \_\_getattribute\_\_ 和 \_\_setattr\_\_ magic method，多檢查預存的_brush實例裡有沒有想要的attribute。

```Python
class Pencil:
    def draw_line():
        print('Draw line with pencil')

class MarkPen:
    def draw_line():
        print('Draw line with MarkPen')

def choose_brush(brush_name):
    if brush_name == 'pencil'
        return Pencil
    else:
        return MarkPen   

class Brush:
    def __init__(self, brush_name)
        self._brush = choose_brush(brush_name)()
        
    def __getattribute__(self, item):
        try:
            return super(Brush, self).__getattribute__(item)
        except AttributeError as e:
            try:
                return getattr(super(Brush, self).__getattribute__('_brush'), item)
            except AttributeError:
                raise e

    def __setattr__(self, item, value):
        try:
            getattr(super(Brush, self).__getattribute__('_brush'), item)
            super(Brush, self).__getattribute__('_brush').__setattr__(item, value)
        except AttributeError:
            super(Brush, self).__setattr__(item, value)
    
    def draw_line(self):
        print('You have a brush.')
        self._brush.draw_line() # Safe
        
    def draw_2_line(self)
        self.draw_line()
        self.draw_line()
```
雖然這樣寫實際上是創造一個實例，並調用實例裡的方法，而不是繼承原本的class，但在操作上可以當作是繼承了一個父類別。

可以進一步用一個中間的 Forking class 來讓各個 class 的目的更完整：

```Python
...

class _Basicbrush:
    def __init__(self, brush_name)
        self._brush = choose_brush(brush_name)()
        
    def __getattribute__(self, item):
        try:
            return super(_Basicbrush, self).__getattribute__(item)
        except AttributeError as e:
            try:
                return getattr(super(_Basicbrush, self).__getattribute__('_brush'), item)
            except AttributeError:
                raise e

    def __setattr__(self, item, value):
        try:
            getattr(super(_Basicbrush, self).__getattribute__('_brush'), item)
            super(_Basicbrush, self).__getattribute__('_brush').__setattr__(item, value)
        except AttributeError:
            super(_Basicbrush, self).__setattr__(item, value)
    

class Brush(_Basicbrush):
    def __init__(self, brush_name)
        super().__init__(self, brush_name)
    
    def draw_line(self):
        print('You have a brush.')
        self._brush.draw_line() # Safe
        
    def draw_2_line(self)
        self.draw_line()
        self.draw_line()
```
寫成這樣以後，想要調用或修改"父類別"中的method或是instance都可以。然而這樣在使用時與真實的繼承機制還是有一點點差別，那就是在調用"父類別"的方法時使用本來的方法是不行的：
```Python
    #In class Brush
    def draw_line(self):
        # super().draw_line()         # Fail
        # _Basicbrush.draw_line(self) # Fail
        self._brush.draw_line()       # Safe
```
要將這點Workaround過去其實也是可行的
只要在_BasicBrush裡面重新map一次共用的方法就好
```Python
    #In class _BasicBrush
    def draw_line(self):
        self._brush.draw_line()
        
    def _maybe_other_method(self):
        self._brush._maybe_other_method()
```

## 懶人病
最後，為了達到懶惰的極致，我把最後的這個workaround包成了一個[函式](https://github.com/howish/python_tools/blob/master/class_structure/forking_class.py)，這讓我以後想做類似的事情時，只要輸入必要的資訊
```Python
_BasicBrush = create_forked_class(
    '_BasicBrush',   # Class name
    '_brush',        # Instance name
    '_brush_name',   # Flag name
    { 
        'Pencil': Pencil,
        'MarkPen': MarkPen,
    },               # Super class options
    ('draw_line', )  # Methods inherited 
)
```
就直接完成了以上的workaround，想直接調用或是繼承都可以啦！
```Python
class MySuperBrush(_BasicBrush):
    def draw_line(self):
        print('Draw a Rocket')
        super().draw_line()
    
    def fire_a_rocket(self, num):
        for _ in range(num):
            self.draw_line()

# Usage
msb = MySuperBrush('pencil')
msb.fire_a_rocket(3)
```
運行結果大概如下
```bash
~ python my_super_brush_with_pencil.py
Draw a Rocket
Draw line with Pencil
Draw a Rocket
Draw line with Pencil
Draw a Rocket
Draw line with Pencil
```
## 結論

雖然達到了目的，但實際上都只是一些workaround而已。充其量只能算是「模擬」了在runtime時選擇要繼承的父類別。不知道是否能有實際意義上的繼承方法呢？

**Unittest**
setUp 和 tearDown 是 `unittest.TestCase` 类中用于管理测试前置准备和后置清理的钩子方法，由测试框架在每个测试方法执行前后自动调用。
```python
import unittest

class TestDemo(unittest.TestCase):
    def setUp(self):
        self.data = [1, 2, 3]
        print('setup')

    def tearDown(self):
        print('tearDown')

    def test_add(self):
        self.assertEqual(sum(self.data), 6)

    def test_contains(self):
        self.assertIn(2, self.data)

if __name__ == '__main__':  
    unittest.main()
```

**pytest**
 fixture 是 pytest 提供的测试前置/后置管理机制，通过 `@pytest.fixture` 装饰器将普通函数标记为"fixture"，测试函数通过在参数列表中声明 fixture 名称来自动获取其返回值。
```python
import pytest

@pytest.fixture
def data():
    print('\nsetup')
    yield [1, 2, 3]
    print('\nteardown')

def test_add(data):
    assert sum(data) == 6

def test_contains(data):
    assert 2 in data
```
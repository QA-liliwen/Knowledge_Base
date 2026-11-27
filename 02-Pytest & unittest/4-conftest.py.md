`conftest.py` 是个特殊文件名，pytest 看到这个名字就自动加载里面的 fixture。同目录下所有测试文件不用import 就能用。相当于一个公共工具箱，放进去的东西大家都能拿。

**conftest.py**
```py
@pytest.fixture  
def shared_token():    
	print("  >> [conftest] generating token...")    return "token_abc_123"
```

**test_use_conftest.py**
```py
# NOTE: no import of conftest here! But shared_token still works.  
def test_login(shared_token):
    print(f"  >> [test1] using token: {shared_token}")
	assert shared_token.startswith("token")
```

**测试结果**
```
test_login  
  >> [conftest] generating token...         ← conftest 里的 fixture 自动执行  >> [test1] using token: token_abc_123     ← 测试拿到了值PASSED  
  
test_profile  
  >> [conftest] generating token...  
  >> [test2] using token again: token_abc_123  
PASSED
```
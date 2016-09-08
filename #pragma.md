```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-retain-cycles"
.... //一些代码
#pragma clang diagnostic pop
```

用来关闭警告的，写一些代码的时候需要忽略某些警告.
```
#pragma mark - UITableViewDataSources
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wunused-variable"
    int unusedInt;
#pragma clang diagnostic pop
    return 10;
}
```

**[Which Clang Warning Is Generating This Message?](http://fuckingclangwarnings.com)**

关闭警告：
```
NSString *unsedString;  
#pragma unused(unsedString)
```


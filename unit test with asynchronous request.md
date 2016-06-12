# unit test with asynchronous request
```objc
- (void)testLogin
{
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [[miiiCasaServer getServer] login:@"test@example.com" andPassword:@"password" success:^(AFHTTPRequestOperation *operation, id responseObject) {
        assertThat(responseObject[@"status"], is(@"ok"));
        dispatch_semaphore_signal(semaphore);
    } failure:nil];
    while (dispatch_semaphore_wait(semaphore, DISPATCH_TIME_NOW))
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                                 beforeDate:[NSDate dateWithTimeIntervalSinceNow:10]];
    dispatch_release(semaphore);   // You don't need this if your deployment target >= 6.0 and ARC enabled.
}
- (void)testLoginFail
{
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    [[miiiCasaServer getServer] login:@"test@example.com" andPassword:@"wrongpassword" success:nil failure:^(AFHTTPRequestOperation *operation, NSError *error) {
        assertThat([[error userInfo] objectForKey:@"errmsg"], containsString(@"incorrect"));
        assertThatInteger([error code], equalToInt(401));
        dispatch_semaphore_signal(semaphore);
    }];
    while (dispatch_semaphore_wait(semaphore, DISPATCH_TIME_NOW))
        [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode
                                 beforeDate:[NSDate dateWithTimeIntervalSinceNow:10]];
    dispatch_release(semaphore);   // You don't need this if your deployment target >= 6.0 and ARC enabled.
}
```

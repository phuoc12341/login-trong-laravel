# Login trong laravel

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Khi chúng ta gõ username + password xong và ấn vào nút Login, thì trong Laravel sẽ gọi đến 1 function login() như thế này

```
    /**
     * Handle a login request to the application.
     *
     * @param  \App\Http\Requests\Request $request
     * @return \Illuminate\Http\RedirectResponse|\Illuminate\Http\Response|\Illuminate\Http\JsonResponse
     *
     * @throws \Illuminate\Validation\ValidationException
     */
    public function login(Request $request)
    {
        if (method_exists($this, 'hasTooManyLoginAttempts') &&
            $this->hasTooManyLoginAttempts($request)) {
            $this->fireLockoutEvent($request);

            return $this->sendLockoutResponse($request);
        }

        if ($this->attemptLogin($request)) {
            $this->sendLoginResponse($request);
        }

        $this->incrementLoginAttempts($request);

        return $this->sendFailedLoginResponse($request);
    }
```
    
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ở đây, hàm login() nó cho ta biết ta có sử dụng chức năng LoginAttempts hay không, nếu dùng thì ta định nghĩa hàm này như sau:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; LoginAttempts: Nỗ lưc login(), chức năng có tác dụng nếu ta login sai nhiều lần thì ta sẽ bị block trong 1 khoảng thời gian

```
    /**
     * Determine if the user has too many failed login attempts.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return bool
     */
    public function hasTooManyLoginAttempts(Request $request)
    {
        return $this->tooManyAttempts(
            $this->throttleKey($request), $this->maxAttempts()
        );
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; function hasTooManyLoginAttempts() để check xem user có login sai nhiều lần hay không
```
     * Determine if the given key has been "accessed" too many times.
     *
     * @param  string  $key
     * @param  int  $maxAttempts
     * @return bool
     */
    public function tooManyAttempts($key, $maxAttempts)
    {
        if ($this->attempts($key) >= $maxAttempts) {
            if ($this->cache->has($key.':timer')) {
                return true;
            }

            $this->resetAttempts($key);
        }

        return false;
    }
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Mỗi user sẽ được xác định thông qua 1 cái key, và ta sẽ xác định user này login bao nhiều lần thông qua cái key này
```
    /**
     * Get the number of attempts for the given key.
     *
     * @param  string  $key
     * @return mixed
     */
    public function attempts($key)
    {
        return $this->cache->get($key, 0);
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Đầu tiên ta check $key tương ứng với user hiện đã nỗ lực login mấy lần trong cache, mặc định sẽ là 0. Sau đó so sánh với 1 giá trị $maxAttempts (giá trị này cho phép user nhập sai bao nhiêu lần nhập sai liên tiếp, mặc định là 5)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Mặc định cache mà Laravel sẽ sử dụng là Illuminate\Contracts\Cache\Repository as Cache
```
    /**
     * The cache store implementation.
     *
     * @var \Illuminate\Contracts\Cache\Repository
     */
    protected $cache;

    public function __construct(Cache $cache)
    {
        $this->cache = $cache;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu user nhập sai vượt quá số lần cho phép thì Laravel sẽ reset lại số lần nỗ lực của user trong Cache nh
```
    /**
     * Reset the number of attempts for the given key.
     *
     * @param  string  $key
     * @return mixed
     */
    public function resetAttempts($key)
    {
        return $this->cache->forget($key);
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  resetAttempts(): reset lại số lần nỗ lực của user trong Cache

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Mặc định cache mà Laravel sẽ sử dụng là Illuminate\Contracts\Cache\Repository as Cache
```
     /**
     * The cache store implementation.
     *
     * @var \Illuminate\Contracts\Cache\Repository
     */
    protected $cache;

    public function __construct(Cache $cache)
    {
        $this->cache = $cache;
    }
```
    
```
    /**
     * Get the throttle key for the given request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return string
     */
    public function throttleKey(Request $request)
    {
        return Str::lower($request->input($this->username())).'|'.$request->ip();
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Còn đây là định dạng "key" - tương ứng với user sẽ được lưu trong Cache: username!ip. Ví dụ: keu.minh.phuoc!10.10.1.2 
```
    /**
     * Get the maximum number of attempts to allow.
     *
     * @return int
     */
    public function maxAttempts()
    {
        return property_exists($this, 'maxAttempts') ? $this->maxAttempts : 5;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu chúng ta không định nghĩa số lần tối đa cho phép nhập sai thì Laravel sẽ lấy mặc định là 5
```
    /**
     * Fire an event when a lockout occurs.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return void
     */
    public function fireLockoutEvent(Request $request)
    {
        event(new Lockout($request));
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Khi tài khoản bị khóa thì sẽ có 1 sự kiện "fireLockoutEvent" được sinh ra 
```
    /**
     * Redirect the user after determining they are locked out.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return void
     *
     * @throws \Illuminate\Validation\ValidationException
     */
    protected function sendLockoutResponse(Request $request)
    {
        $seconds = $this->availableIn(
            $this->throttleKey($request)
        );

            throw ValidationException::withMessages([
            $this->username() => [__('auth.throttle', [
                'seconds' => $seconds,
                'minutes' => ceil($seconds / 60),
            ])],
        ])->status(Response::HTTP_TOO_MANY_REQUESTS);
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Sau đó Laravel sẽ gửi 1 response Lockout cho user, điểm chú ý là thời gian user bị block (trong thông báo gửi về) chính là thời gian của bộ đếm ngược lưu trong Cache 
```
    /**
     * Get the number of seconds until the "key" is accessible again.
     *
     * @param  string  $key
     * @return int
     */
    public function availableIn($key)
    {
        return $this->cache->get($key.':timer') - $this->currentTime();
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Bao lâu nữa user có thể login lại = (thời điểm user có thể login lại) - (thời gian hiện tại)
```
    /**
     * Get the current system time as a UNIX timestamp.
     *
     * @return int
     */
    protected function currentTime()
    {
        return Carbon::now()->getTimestamp();
    }
```
```
    /**
     * Attempt to log the user into the application.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return bool
     */
    public function attemptLogin(Request $request)
    {
        return $this->guard()->attempt($this->credentials($request), $request->filled('remember'));
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  Việc check username + password sẽ do hàm attemptLogin() đảm nhận, thông qua Guard login()
```
    /**
     * Get the needed authorization credentials from the request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array
     */
    protected function credentials(Request $request)
    {
        return $request->only($this->username(), 'password');
    }
```
```
    public function username()
    {
        return 'email';
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;  Hàm này cho phép ta login = username hoặc email -> tùy ta định nghĩa thế nào
```
    /**
     * Send the response after the user was authenticated.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\Response
     */
    protected function sendLoginResponse(Request $request)
    {
        $request->session()->regenerate();

        $this->clearLoginAttempts($request);

        return $this->authenticated($request, $this->guard()->user())
                ?: redirect()->intended($this->redirectPath());
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu user login thành công thì Laravel sẽ gửi về 1 response login kèm với 1 session
```
    /**
     * Clear the login locks for the given user credentials.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return void
     */
    function clearLoginAttempts(Request $request)
    {
        $this->clear($this->throttleKey($request));
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Nếu login thành công thì sẽ xóa throttleKey trong Cache và lockout timeer đi, coi như user chưa nhập sai lần nào
```
    /**
     * Clear the hits and lockout timer for the given key.
     *
     * @param  string  $key
     * @return void
     */
    public function clear($key)
    {
        $this->resetAttempts($key);

        $this->cache->forget($key.':timer');
    }
```
```
    /**
     * The user has been authenticated.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  mixed  $user
     * @return mixed
     */
    protected function authenticated(Request $request, $user)
    {
        //
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Laravel dành sẵn 1 nơi ta định nghĩa thêm bất cứ thứ gì khi user login thành công
```
    /**
     * Increment the login attempts for the user.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return void
     */
    public function incrementLoginAttempts(Request $request)
    {
        $this->hit($this->throttleKey($request), $this->decayMinutes() * 60);
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; TH user login sai, thì ta sẽ tăng số lần nỗ lực login của user lên
```
    /**
     * Get the number of minutes to throttle for.
     *
     * @return int
     */
    public function decayMinutes()
    {
        return property_exists($this, 'decayMinutes') ? $this->decayMinutes : 1;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; lấy ra số phút user bị block, mặc định sẽ là 1 phút nếu ta ko định nghĩa gì
```
    /**
     * Increment the counter for a given key for a given decay time.
     *
     * @param  string  $key
     * @param  int  $decaySeconds
     * @return int
     */
    public function hit($key, $decaySeconds = 60)
    {
        $this->cache->put($key.':timer', $this->availableAt($decaySeconds), $decaySeconds);
        $added = $this->cache->add($key, 0, $decaySeconds);
        $hits = (int) $this->cache->increment($key);
        if (! $added && $hits == 1) {
            $this->cache->put($key, 1, $decaySeconds);
        }

        return $hits;
    }
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Ở đây Laravel sẽ thêm hoặc cập nhật bộ định thời vào Cache: chỉ ra thời điểm user có thể login lại

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Đồng thời tăng số lần nỗ lực của user lên 1

```
    /**
     * Get the "available at" UNIX timestamp.
     *
     * @param  \DateTimeInterface|\DateInterval|int  $delay
     * @return int
     */
    protected function availableAt($delay = 0)
    {
        $delay = $this->parseDateInterval($delay);

        return $delay instanceof DateTimeInterface
                            ? $delay->getTimestamp()
                            : Carbon::now()->addRealSeconds($delay)->getTimestamp();
    }

    /**
     * If the given value is an interval, convert it to a DateTime instance.
     *
     * @param  \DateTimeInterface|\DateInterval|int  $delay
     * @return \DateTimeInterface|int
     */
    protected function parseDateInterval($delay)
    {
        if ($delay instanceof DateInterval) {
            $delay = Carbon::now()->add($delay);
        }

        return $delay;
    }

    /**
     * Get the failed login response instance.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Symfony\Component\HttpFoundation\Response
     *
     * @throws \Illuminate\Validation\ValidationException
     */
    protected function sendFailedLoginResponse(Request $request)
    {
        throw ValidationException::withMessages([
            $this->username() => [ __('auth.failed') ],
        ]);
    }
    
```

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; Cuối cùng nếu user login ko thành công sẽ có 1 failed login response được server trả về 

Như vậy là ta đã biết toàn bộ luồng xử lý Login trong Laravel rồi, xin chào các bạn !!
    
    

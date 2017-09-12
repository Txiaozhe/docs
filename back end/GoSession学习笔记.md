## Go如何使用session

通过上一小节的介绍，我们知道 session 是在服务器端实现的一种用户和服务器之间认证的解决方案，目前 Go 标准包没有为 session 提供任何支持，这小节我们将会自己动手来实现 go 版本的 session 管理和创建

### session创建过程

session 的基本原理是由服务器为每个会话维护一份信息数据，客户端和服务端依靠一个全局唯一的标示来访问这份数据，以达到交互的目的。当用户访问 Web 应用时，服务端程序会随需要创建 session ，这个过程可以概括为三个步骤：

- 生成全局唯一标识符（sessionid）；
- 开辟数据存储空间。一般会在内存中创建相应的数据结构，但这种情况下，系统一旦掉电，所有的会话数据就会丢失，如果是电子商务类网站，这将造成严重的后果。所以为了解决这类问题，你可以将会话数据写到文件里或存储在数据库中，当然这样会增加I/O开销，但是它可以实现某种程度的 session 持久化，也更有利于 session 的共享；
- 将 session 的全局唯一标示符发送给客户端。

以上三个步骤中，最关键的是如何发送这个 session 的唯一标示这一步上。考虑到 HTTP 协议的定义，数据无非可以放到请求行、头域或 Body 里，所以一般来说会有两种常用的方式：cookie 和 URL 重写。

1. Cookie 服务端通过设置 Set-cookie 头就可以将session 的标识符传送到客户端，而客户端此后的每一次请求都会带上这个标识符，另外一般包含session 信息的 cookie 会将失效时间设置为0(会话cookie)，即浏览器进程有效时间。至于浏览器怎么处理这个0，每个浏览器都有自己的方案，但差别都不会太大(一般体现在新建浏览器窗口的时候)；
2. URL 重写 所谓 URL 重写，就是在返回给用户的页面里的所有的 URL 后面追加 session 标识符，这样用户在收到响应之后，无论点击响应页面里的哪个链接或提交表单，都会自动带上 session 标识符，从而就实现了会话的保持。虽然这种做法比较麻烦，但是，如果客户端禁用了 cookie 的话，此种方案将会是首选。

### Go实现session管理

通过上面 session 创建过程的讲解，读者应该对 session邮轮一个大体的认识，但是具体到动态页面技术里面，又是怎么实现 session 的呢？下面我们将结合 session 的生命周期（lifecycle)，来实现go语言版本的 session 管理。

#### session管理设计

我们知道 session 管理涉及到如下几个因素

- 全局session 管理器
- 保证sessionid 的全局唯一性
- 为每个客户关联一个session
- session 的存储(可以存储到内存、文件、数据库等)
- session 过期处理

接下来我将讲解一下我关于 session 管理的整个设计思路以及相应的 go 代码示例：

#### Session管理器

定义一个全局的 session 管理器

```go
    type Manager struct {
        cookieName  string     //private cookiename
        lock        sync.Mutex // protects session
        provider    Provider
        maxlifetime int64
    }

    func NewManager(provideName, cookieName string, maxlifetime int64) (*Manager, error) {
        provider, ok := provides[provideName]
        if !ok {
            return nil, fmt.Errorf("session: unknown provide %q (forgotten import?)", provideName)
        }
        return &Manager{provider: provider, cookieName: cookieName, maxlifetime: maxlifetime}, nil
    }
```

Go实现整个的流程应该也是这样的，在 mian 包中创建一个全局的 session 管理器

```go
    var globalSessions *session.Manager
    //然后在init函数中初始化
    func init() {
        globalSessions, _ = NewManager("memory","gosessionid",3600)
    }
```

我们知道 session 是保存在服务器端的数据，它可以以任何的方式存储，比如存储在内存、数据库或者文件中。因此我们抽象出一个 Provider 接口，用以表征 session 管理器底层存储结构。

```go
    type Provider interface {
        SessionInit(sid string) (Session, error)
        SessionRead(sid string) (Session, error)
        SessionDestroy(sid string) error
        SessionGC(maxLifeTime int64)
    }
```

- SessionInit 函数实现 Session 的初始化，操作成功则返回此新的Session变量
- SessionRead 函数返回 sid 所代表的 Session 变量，如果不存在，那么将以sid为参数调用 SessionInit 函数创建并返回一个新的 Session 变量
- SessionDestroy 函数用来销毁 sid 对应的 Session 变量
- SessionGC 根据 maxLifeTime 来删除过期的数据

那么 Session 接口需要实现什么样的功能呢？有过 Web开发经验的读者知道，对 Session 的处理基本就 设置值、读取值、删除值以及获取当前 sessionID 这四个操作，所以我们的 Session 接口也就实现这四个操作。

```go
    type Session interface {
        Set(key, value interface{}) error //set session value
        Get(key interface{}) interface{}  //get session value
        Delete(key interface{}) error     //delete session value
        SessionID() string                //back current sessionID
    }
```

```
    }
```

> 以上设计思路来源于 database/sql/driver，先定义好接口，然后具体的存储 session 的结构实现相应的接口并注册后，相应功能这样就可以使用了，以下是用来随需注册存储 session 的结构的 Register 函数的实现。

```Go
    var provides = make(map[string]Provider)

    // Register makes a session provide available by the provided name.
    // If Register is called twice with the same name or if driver is nil,
    // it panics.
    func Register(name string, provider Provider) {
        if provider == nil {
            panic("session: Register provide is nil")
        }
        if _, dup := provides[name]; dup {
            panic("session: Register called twice for provide " + name)
        }
        provides[name] = provider
    }
```

#### 全局唯一的Session ID

Session ID是用来识别访问 Web 应用的每一个用户，因此必须保证它是全局唯一的（GUID）， 下面代码展示了如何满足这一需求：

```go
    func (manager *Manager) sessionId() string {
        b := make([]byte, 32)
        if _, err := io.ReadFull(rand.Reader, b); err != nil {
            return ""
        }
        return base64.URLEncoding.EncodeToString(b)
    }
```

#### Session创建

我们需要为每个来访用户分配或获取与他相关联的Session，以便以后根据Session信息来验证操作。SessionStart 这个函数就是用来检测是否已经有某个Session 与当前来访用户发生了关联，如果没有则创建之。

```go
    func (manager *Manager) SessionStart(w http.ResponseWriter, r *http.Request) (session Session) {
        manager.lock.Lock()
        defer manager.lock.Unlock()
        cookie, err := r.Cookie(manager.cookieName)
        if err != nil || cookie.Value == "" {
            sid := manager.sessionId()
            session, _ = manager.provider.SessionInit(sid)
            cookie := http.Cookie{Name: manager.cookieName, Value: url.QueryEscape(sid), Path: "/", HttpOnly: true, MaxAge: int(manager.maxlifetime)}
            http.SetCookie(w, &cookie)
        } else {
            sid, _ := url.QueryUnescape(cookie.Value)
            session, _ = manager.provider.SessionRead(sid)
        }
        return
    }
```

我们用 login 操作来演示 session 的运用：

```go
    func login(w http.ResponseWriter, r *http.Request) {
        sess := globalSessions.SessionStart(w, r)
        r.ParseForm()
        if r.Method == "GET" {
            t, _ := template.ParseFiles("login.gtpl")
            w.Header().Set("Content-Type", "text/html")
            t.Execute(w, sess.Get("username"))
        } else {
            sess.Set("username", r.Form["username"])
            http.Redirect(w, r, "/", 302)
        }
    }
```

#### 设置、读取和删除

SessionStart 函数返回的是一个满足 Session 接口的变量，那么我们该如何用他来对 Session 数据进行操作呢？

上年的例子中的代码 `session.Get("uid")` 已经展示了基本的读取数据的操作，现在我们再来看一下详细的操作：

```go
    func count(w http.ResponseWriter, r *http.Request) {
        sess := globalSessions.SessionStart(w, r)
        createtime := sess.Get("createtime")
        if createtime == nil {
            sess.Set("createtime", time.Now().Unix())
        } else if (createtime.(int64) + 360) < (time.Now().Unix()) {
            globalSessions.SessionDestroy(w, r)
            sess = globalSessions.SessionStart(w, r)
        }
        ct := sess.Get("countnum")
        if ct == nil {
            sess.Set("countnum", 1)
        } else {
            sess.Set("countnum", (ct.(int) + 1))
        }
        t, _ := template.ParseFiles("count.gtpl")
        w.Header().Set("Content-Type", "text/html")
        t.Execute(w, sess.Get("countnum"))
    }
```

#### Session重置

我们知道，Web 应用中用户退出这个操作，那么当用户退出应用的时候，我们需要对该用户的 session 数据进行销毁操作，下面这个函数就是实现了这个功能：

```Go
    //Destroy sessionid
    func (manager *Manager) SessionDestroy(w http.ResponseWriter, r *http.Request){
        cookie, err := r.Cookie(manager.cookieName)
        if err != nil || cookie.Value == "" {
            return
        } else {
            manager.lock.Lock()
            defer manager.lock.Unlock()
            manager.provider.SessionDestroy(cookie.Value)
            expiration := time.Now()
            cookie := http.Cookie{Name: manager.cookieName, Path: "/", HttpOnly: true, Expires: expiration, MaxAge: -1}
            http.SetCookie(w, &cookie)
        }
    }
```

#### Session销毁

我们来看一下 Session 管理器如何来管理销毁，只要我们在 Main 启动的时候启动：

```go
    func init() {
        go globalSessions.GC()
    }

    func (manager *Manager) GC() {
        manager.lock.Lock()
        defer manager.lock.Unlock()
        manager.provider.SessionGC(manager.maxlifetime)
        time.AfterFunc(time.Duration(manager.maxlifetime), func() { manager.GC() })
    }
```

我们可以看到 GC 充分利用了 time 包中的定时器功能，当超时 `maxLifeTime` 之后调用 GC 函数，这样就可以保证 `maxLifeTime` 时间内  Session 都是可用的， 类似的方案也可以用于统计在线用户数之类的。

### Session存储

示例一个基于 __内存__ 的session存储接口的实现，代码如下：

```go
package memory

    import (
        "container/list"
        "github.com/astaxie/session"
        "sync"
        "time"
    )

    var pder = &Provider{list: list.New()}

    type SessionStore struct {
        sid          string                      //session id唯一标示
        timeAccessed time.Time                   //最后访问时间
        value        map[interface{}]interface{} //session里面存储的值
    }

    func (st *SessionStore) Set(key, value interface{}) error {
        st.value[key] = value
        pder.SessionUpdate(st.sid)
        return nil
    }

    func (st *SessionStore) Get(key interface{}) interface{} {
        pder.SessionUpdate(st.sid)
        if v, ok := st.value[key]; ok {
            return v
        } else {
            return nil
        }
        return nil
    }

    func (st *SessionStore) Delete(key interface{}) error {
        delete(st.value, key)
        pder.SessionUpdate(st.sid)
        return nil
    }

    func (st *SessionStore) SessionID() string {
        return st.sid
    }

    type Provider struct {
        lock     sync.Mutex               //用来锁
        sessions map[string]*list.Element //用来存储在内存
        list     *list.List               //用来做gc
    }

    func (pder *Provider) SessionInit(sid string) (session.Session, error) {
        pder.lock.Lock()
        defer pder.lock.Unlock()
        v := make(map[interface{}]interface{}, 0)
        newsess := &SessionStore{sid: sid, timeAccessed: time.Now(), value: v}
        element := pder.list.PushBack(newsess)
        pder.sessions[sid] = element
        return newsess, nil
    }

    func (pder *Provider) SessionRead(sid string) (session.Session, error) {
        if element, ok := pder.sessions[sid]; ok {
            return element.Value.(*SessionStore), nil
        } else {
            sess, err := pder.SessionInit(sid)
            return sess, err
        }
        return nil, nil
    }

    func (pder *Provider) SessionDestroy(sid string) error {
        if element, ok := pder.sessions[sid]; ok {
            delete(pder.sessions, sid)
            pder.list.Remove(element)
            return nil
        }
        return nil
    }

    func (pder *Provider) SessionGC(maxlifetime int64) {
        pder.lock.Lock()
        defer pder.lock.Unlock()

        for {
            element := pder.list.Back()
            if element == nil {
                break
            }
            if (element.Value.(*SessionStore).timeAccessed.Unix() + maxlifetime) < time.Now().Unix() {
                pder.list.Remove(element)
                delete(pder.sessions, element.Value.(*SessionStore).sid)
            } else {
                break
            }
        }
    }

    func (pder *Provider) SessionUpdate(sid string) error {
        pder.lock.Lock()
        defer pder.lock.Unlock()
        if element, ok := pder.sessions[sid]; ok {
            element.Value.(*SessionStore).timeAccessed = time.Now()
            pder.list.MoveToFront(element)
            return nil
        }
        return nil
    }

    func init() {
        pder.sessions = make(map[string]*list.Element, 0)
        session.Register("memory", pder)
    }
```

上面这个代码实现了一个内存存储的 session 机制。通过 init 函数注册到session管理器中。这样就可以方便的调用了。我们如何来调用该引擎呢？请看下面的代码

```go
    import (
        "github.com/astaxie/session"
        _ "github.com/astaxie/session/providers/memory"
    )
```

当 import 的时候已经执行了 memory 函数里面的 init 函数，这样就已经注册到 session 管理器中，我们就可以使用了，通过该如下方式就可以初始化一个session管理器：

```go
    var globalSessions *session.Manager

    //然后在init函数中初始化
    func init() {
        globalSessions, _ = session.NewManager("memory", "gosessionid", 3600)
        go globalSessions.GC()
    }
```

### 预防session劫持

session 劫持是一种广泛存在的比较严重的安全威胁，在 session 技术中，客户端和服务端通过 session 的标识符来维护会话，但这个标识符很容易就能被嗅探到，从而被其他人利用进行中间人攻击的一种类型。

#### cookieonly和token

如何有效的防止session劫持呢？

其中一个解决方案就是sessionID的值只允许cookie设置，而不是通过URL重置方式设置，同时设置cookie的httponly为true,这个属性是设置是否可通过客户端脚本访问这个设置的cookie，第一这个可以防止这个cookie被XSS读取从而引起session劫持，第二cookie设置不会像URL重置方式那么容易获取sessionID。

第二步就是在每个请求里面加上token，实现类似前面章节里面讲的防止form重复递交类似的功能，我们在每个请求里面加上一个隐藏的token，然后每次验证这个token，从而保证用户的请求都是唯一性。

​        **详见防止重复提交**

#### 间隔生成新的SID

还有一个解决方案就是，我们给session额外设置一个创建时间的值，一旦过了一定的时间，我们销毁这个sessionID，重新生成新的session，这样可以一定程度上防止session劫持的问题。

```go
    createtime := sess.Get("createtime")
    if createtime == nil {
        sess.Set("createtime", time.Now().Unix())
    } else if (createtime.(int64) + 60) < (time.Now().Unix()) {
        globalSessions.SessionDestroy(w, r)
        sess = globalSessions.SessionStart(w, r)
    }
```

session启动后，我们设置了一个值，用于记录生成sessionID的时间。通过判断每次请求是否过期(这里设置了60秒)定期生成新的ID，这样使得攻击者获取有效sessionID的机会大大降低。
#### 创建 update 的方式
- HostRoot: `ReactDOM.render`
- ClassComponent: `this.setState`
- ClassComponent: `this.forceUpdate`
- FunctionComponent: `useState`
- FunctionComponent: `useReducer`

流程：
```
触发状态更新（根据场景调用不同方法）

    |
    |
    v

创建Update对象

    |
    |
    v

scheduleUpdateOnFiber

    |
    |
    v

从fiber到root（`markUpdateLaneFromFiberToRoot`）

    |
    |
    v

调度更新（`ensureRootIsScheduled`）

    |
    |
    v

render 阶段（`performSyncWorkOnRoot` 或 `performConcurrentWorkOnRoot`）

    |
    |
    v

commit阶段（`commitRoot`）
```
<iframe xmlns="http://www.w3.org/1999/xhtml" frameborder="0" style="width:100%;height:684px;"
    src="https://viewer.diagrams.net/?highlight=0000ff&edit=_blank&layers=1&nav=1&title=react-design.drawio#R%3Cmxfile%20pages%3D%222%22%3E%3Cdiagram%20id%3D%2258Rk17BsqPwjuQky5jl_%22%20name%3D%22%E6%9B%B4%E6%96%B0%E8%B0%83%E7%94%A8%E6%A0%88%22%3E7V1bc6M4Fv41VM0%2BjIuLuD3ajjO7VT3bvZPpmp6nLWJkmwlGLsC5zK8fCSQDEraJg4xI0g9dIG7m0%2FnO%2Bc6RRDRrvn3%2BJQ12m19RCGPN1MNnzbrRTNNzffw%2FaXgpGxzdKRvWaRSWTUbVcBf9DWmjTlv3UQizxok5QnEe7ZqNS5QkcJk32oI0RU%2FN01Yobj51F6yh0HC3DGKx9Y8ozDdlq6XrenXg3zBab3L%2ByDZgZ9OGbBOE6KnWZC00a54ilJdb2%2Bc5jAl4DJjyutsjRw%2B%2FLIVJ3uUCC1hT0%2FJXy293%2F3%2F43wP6nmX%2F%2Fdkq7%2FIYxHv6xvTH5i8MAhhiROguSvMNWqMkiBdV6yxF%2BySE5DE63qvO%2BYLQDjcauPEvmOcvtHuDfY5w0ybfxvSo%2BCr07TK0T5fwxO9nNhGka5ifOI%2B%2BFnmX2gMoUL9AtIV5%2BoJPSGEc5NFjs%2FcDakTrw3kVzHiDIv0K1A0R9YWjeb42dbWFp00NzZuTFn%2BmzUCx4WgzvdiwNO9GWwBtdqP5%2BOFOjN96dp%2FirTXZ0ha32myueZ62sLUpvmeBIszv8iCH7KAv9HHVg6Q7njZRDu92QYH8E%2BZ1x956hGkOn0%2Fiy44CSgrqFkzGmqeKZAYjzqbGL0eX1CXmeSKcgaXJgixP0cPBZ2AoZmGQbQ4Q94Ch14TQAiKEJmiB0PBlYQgEDLMlfuV9DL%2FvQmx%2FX5Pb6B6mJ6xPP299DRhFlAcEFsjC1RZw3QbpQ4nplyCBtynaFsj%2Bjn4j4eRN%2BDYh7clWfbsJqd8CqdECqfV6SL9nMP16%2FxcRA6YeB%2FcwriAj6JSwhQ0tQV9hglsK3zrTvAVxoMTJWmQDu0zsdrF%2BKO1XLw4uNK883dO8KWnBb1U63pmvzZzi8QksH3Syf50adtQeVO49YMnrPbxb68DXhFSRJONSMm5HJWMYV5Iyx4gEk2yfQkKl%2F2R31L%2BfoBNWLi5hFGHRVJvqrbwqiVIcwwzCUgf%2FfPyAyQ4mYZSsiZPLsMTRTMsgHWOyW%2FMCCWueW8I%2B8oyp5gOqlIhkcjV%2FqnmAIPeIO%2Bf3aNuFmu7IqOkqSE1PYCY1Dax0F0Ts4r7BG77b8Jg1mDEWeRuWcxSjFLckKCHkXUVxzDUFcbRO8G4MV%2BQOBNcIJ3dT2ryNwrDgfVvXNTu3h746dAQTvS2d5bX0lSlLV%2FhCx1BiuYQ6%2Fm2RlHgkrJn6vgiblKOUZJipBum%2Fj9JfLarFaCOXtA5jDxtRlmJxahp4g6cphph%2Bn4hp6uYpF0MrLVExxCyayO47LGoeYVhE8Wm2eN5F6VvRlRJMLS6a2hKFbseqhCXAdAURC5%2Bj%2FAe9nGz%2FSbYnNt27ea4dunlhOwl%2B3%2BKiia5brIFe6R0aqouLvcbV32CKRVkOU9rYYBAWglNSz61iBW65jQicp%2F39%2BRIixfi88qal6%2F6Ud3Epfq3gpXbCDkVJntXu%2FI001K20SXvb5%2Bq%2BZ853gMXZZfkLKis9vMobDFesMV7RcI2LDJcz2rMmezDIZRxkWbRs2KSoerRicICMD1zFWk27b2t9W2gQRxp2MF2hdHv3kiz%2FQOnD16QsXvEl7Z9YEZGc%2BK%2B%2Bw0bveoePIV0zMmlB2XI%2FqSiHik5HKlqmWlR0jlFxjpLlPiVIdCLk6MjomEOT0RTHSAQQFcvgHLepH6w2j9aWZsjLg02x2HskmKidwV0MrTz7FH3DKt5nm2%2FErT7CxWoFl6U4VSx3czyuEtqV6tJyNwBGF3d%2FJsmb2Qy%2BwD00yAi%2FrZnf5THZ7DqMYvlKxWT2u2u8wy8fwpS4MeLUxkA5Z2jKmeJQwxJtt1Gu6LA4j6DdMg3mugiyG8utjV4HzLZ5G9cFs2We18jEXqtivq7YY9lT13RlXJKvM8Dy6gNiZWacks%2BROPjdDUoWAEck%2BV5ZasmwsMq5QnzRVivFD1SOsajePiv9bLXKMZZYEqikX%2BXiRkBA1xiagK44GeGTgFcjoN2RgI6nFgHFatIqSqJsU5Hvt4KQY6CgPTQFLbGCNNYMwpU4TbkjmGJCK%2BCmWAbhO02B27mmacjLIMSJbn0WV%2BQnDRdjKi1pAN57D7QDBVHWY2eDKOh9NsrbDEIsHD3hpJx0raLlS99VbcTAHT2p3BHLV9BVvrpW38y7aB6YcZieyK3LOTYRTLjAtnXOZiXMBAMdFuYoJmEMEyinYYAoq%2Fvzr%2FIVzOWQypMw4pAbrSt%2FT6L864rUlBUMWwamrWJxC4j6Wgxkwvzdo8tDGsPDZ%2BPfYbLyxLDrIcyY6AY4HcTwDj%2Fy3JxU3IhWVSyqByyjEQ0nnus2I6LvHc54zdznN4SxriPg7MSzYaxmV3aLWbG210U7MTpZXJID%2BAUg5avTy07FOcAt3AE2d6cSG%2BFOr47Apuc2H%2BSeicCApaLtF8iJwLYozWMiDtu05pdynWednGxN1hKbIrHP46uyUphFfwf3xf2IEVP88M3tmWbftJr1aYfCe8XDh17oU7T6p1TavCWZ0WL7zV4y32av7BS0WmUw57qvnw4b37gxX6HoPKwpTzGxYR6p4yrXr%2F0MP2BsD7oo6R3XfuzOI5hq1X5scQST5SZKj1%2FyFaDhJxDY7yBZVsD1s6%2FOqe%2BhFK6H9e9kjpWl%2BAWNwJ%2F4tX9e8459iXe%2BGuYAFctn9vG6z3g0zOU%2BQp6IeSfVn%2BFjltMhW%2Fms%2FnxWf04HAdNzJmz2VlW2mQBTP%2Fzjbtq9HGT5E4Mr1OAQA6zq1uyrtB%2BzQOSImbLiBSLnmOP8GAUip8tnez9d7qfL%2FXS5qrpcsWSiustlPueDutwu3xf9dLm9uFx1HKQ1cZt26ur4df2LvaIzYV%2F0qu4nyw2CCXOx7FmmPrGdWmHl%2FK99zR0kOUqx%2FlE6Sv2nbIP2cfhnBONQ%2FBKJQo6TeY4P6jjFQovQWYqVtPkhN7dtYft1K9qOuBBAxhqV6w9ndsZWWiXQEWcvKf3RAH7Uyht%2B1Z1YP1Cd4uKgQ%2BdRK2l%2FgsQ9%2Bom8vkrSV6j0X4yrNH574%2FsOz%2FtZE%2Bp2ndLgddX%2B15nS4Ir5%2BT1cR4mqw0KOfuFnJ6UFBW%2F0i1nGTLuua1k8tWYSuWKVZY62uxjmUO1hWZ5%2FzuCijH0MfkyizOPEA2BL88%2BKMnZi%2FzCKo9s9BYIriLGL8ZQmxvxPMTZcVPDMjlGBmY0iUYH97hoHyz9H8%2BMHiQ4YYyWnlxo%2BFxSA10K%2F6waFEc4v5Z3YIdQOGBTEovSyZ5kyQHToDKy8VF2sFTNcVRV%2BPMcde3COd1iNqRjHTcNVTvj5on7uOejIp%2FjluMoTgKKeTuESJcsohvNNFIf4dRXkOfaW14vleLf6s%2FDlOBx%2By82vKCTDmYt%2FAA%3D%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%22K1cIes4Yc2qFtbUKwQ81%22%20name%3D%22fiber%20%E6%A0%91%22%3E7Vxbc9o6EP41np4%2BJOO7zSMQknOmzWmneWjz1BFYAZ0ai5FFAv31R7IlfBGhBAzIkJlOx1pLi9HufvvtysRw%2BtPFHQGzyT2OYGzYZrQwnBvDtsOgw%2F7ngmUu8E0%2FF4wJinKRVQge0G8ohKaQzlEE08pEinFM0awqHOEkgSNakQFC8Et12hOOq586A2OoCB5GIFal31FEJ7nUMU2zuPE3ROMJrd%2BZAjlbCNIJiPBLSeQMDKdPMKb51XTRhzHfPLkxzsS%2Bmz4MQTiff%2FoajEn6vRtc5cpu37Jk9R0ITGizqr1QfDe6lDsGI7aBYogJneAxTkA8KKS90Zw8Q67VYgOC50mUjUw2KhZ8xngmpvwHKV0K1wBziploQqexuAsXiP7gy689MXoUyvj1zaI8WMpBQsnyR3nwWGjgw2JZNpLrIpBOVk%2BeUkBol%2FsYEyQ4gVJ2i%2BJ4tTySM0YxSFM0yoViSq6G4F%2Bwj2NMsg3kHsSdyOltaTZh3hTPyQhumueI%2BAFkDDcp9IJ8IrdkKQqEW9xBPIVsU9gEAmNA0XM1VICIuPFqXuFU7EL41Vt8TEDIM4jn4qM%2B%2FPxJIBjRPk4oQAkkhu1%2BMHhU9fhjgSTC009wudY3P4Mhg6iKC4EYjRNuIrbRTJfTe4aEIgYCXXFjiqIoc10CU%2FQbDDN93EQzjBKafWGvZ3g32xtNBg7%2FJLhYB2XiUypgUdl9serKvDZtX6wVKHsltG9tIKH9K%2F86pSn46SllrlK34OohdjeqdMaSUUeFMf2Y7V1vyK%2FG%2FOovFhYZUNrmzZf7j4pdCxDh5nyZIAofZiCLhheWlqrWzgNOovla%2B78t8hQbvmqrVVoTdrI6YvxS5BZL5otJKa345oGCywpet8M%2FyRPWB97L4L6C%2BvXw3m6Y9raEaRlCmsC07ejjKztRgSL7P5bvnSUV2NbH%2BKxmfSxbyjYCLEsTRBpVE5HATS%2Bo4aZX46q1%2BaG7cT67yJ%2Bg0ZRmuyqUzklmMX15iIzaRniI7Uh1kofY%2B%2FEQodr112o9AkvxFJPeoiEk3zIu0lIS4ulHQjSqIt9AM15LHdZZpw6%2FnfTEUqvIvIbksVyqI7VFaquxipEBdSiTlYTUzn5AfXgoXnXRTo8R1jtGNIURnlYYIRWXMQLxAGdesjbl64MOMjyaQIcqNOxJ4Y5A0nzFanmvSOkifeN4f%2FPlvhfj0S%2BUjLlN29tMcmpFjmOv4XH%2BMXmc7WmD0dUWgLzzjtFKZX8CjN6pBVAv6e3O5hZAvWVQm3%2BgFoAKRcwHKPwXR1Dn5CHL3CaaAJZVP4zQPoM4bitx67j4s%2BuZ6alwy278FLMR3HKCI%2BCQo%2FatRhMURxpjkIzBZjDIdMMqBln7YZB8oBqyWea136kqOWAFbCtG5QQ2a0i2lsQGoW7NSMfWNBnsdo513iQ22DYZ2Fokg45fI6XHOJdy1FctCKRzkuicDSTWNdDOMDueXd11V3tCGmiEQfZOlbR9OafpW6OQ4zeNQvtlOnmi0CKWKF9XaqhSreKC7rBgqy9rdWczJjAGgdHrGaFrDFx%2B0fNbSwg7Zq1eck9OCNWmToSez2aD3eDUG%2BxaymZearYrclk54VnVhCdSYpHtzAazXcMMWaXAnZr7mTW3ytOtWLWhwasoCmqK8nSsKGoKjF21HNeeVruvocVFpE9Xp%2BOpS2fVfyTL7mH6tyqOOFUc8ZztAKkxr1Tphe4sXAbSZcKIpw9d2e1NpAqEnPdv3txtS3PRiNOlNPfUA5wUDWOUjDWGBa9JdmF6frVrt%2BcJzhHohVqdpzOg0sG2Foq%2BefJCUaP3xN8LxSMXitvysj8q8usV56ELRfXNc%2F0LxSZ%2Fntw%2BhmdrhDM7HAFbxuUwPEnc2nb44qm%2FFtQeFWRcNHEqa7k1fqc%2FwZOudi4Ez1qxN20YnqdyaH621e3zf%2FxsKzRC0xj4mcQyBp4Rhkanxy%2F4sVeXT%2B50%2BUEYc6Ti5Sjm9betNVPo1a10MCOxYfH3iPKwKf6qkzP4Hw%3D%3D%3C%2Fdiagram%3E%3C%2Fmxfile%3E" />

#### Update

`ClassComponent` 与 `HostRoot`（即 `rootFiber.tag` 对应类型）共用同一种 `Update` 结构，如下：

```ts
type Update<State> = {
  // TODO: Temporary field. Will remove this by storing a map of
  // transition -> event time on the root.
  eventTime: number
  lane: Lane

  tag: UpdateState | ReplaceState | ForceUpdate | CaptureUpdate
  // ClassComponent: 调用 setState 传递的参数，当为函数时，接受 (prevState, nextProps) 参数
  // HostRoot:  { element: ReactDOM.render 的第一个传参 }
  payload: any
  // 更新的回调函数
  // ClassComponent: this.setState 的第二个传参
  // FunctionComponent: 
  // HostRoot： ReactDOM.render 的第三个传参
  callback: (() => mixed) | null

  next: Update<State> | null
};
```

Fiber 上的多个 `Update` 会组成链表并被包含在 `fiber.updateQueue` 中

#### UpdateQueue
`updateQueue` 有三种类型：

**HostComponent**: 
```ts
workInProgress.updateQueue = updatePayload;
```
其中 `updatePayload` 为数组形式，他的偶数索引的值为变化的 `prop key`，奇数索引的值为变化的 `prop value`

**ClassComponent** 与 **HostRoot** 使用的 `UpdateQueue` 结构如下：

```ts
type UpdateQueue<State> = {
  // 本次更新前该 fiber 的 state，Update 基于该 state 计算更新后的 state 
  baseState: State
  firstBaseUpdate: Update<State> | null
  lastBaseUpdate: Update<State> | null
  shared: {
    // 触发更新时，产生的 Update 会保存在 shared.pending 中形成单向环状链表。当由 Update 计算 state 时这个环会被剪开并连接在 lastBaseUpdate 后面
    pending: Update<State> | null
    interleaved: Update<State> | null
  }
  // 有 callback （this.setState 的第二个参数或者 ReactDom.render 的第三个参数） 的 update 对象列表
  // callback 在 commit 的 layout阶段被执行 （通过调用 commitUpdateQueue）
  effects: Array<Update<State>> | null
};
```

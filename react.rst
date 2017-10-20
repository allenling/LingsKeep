react ssr
==========

其实ssr没什么的, 就是django以前渲染html那一套嘛, 只不过是用node然后调用react组件来在node里面渲染react而已

比较坑的是react的机制

基本流程
----------


写react之后, 打成一个node包(也就是编译成es5而已, babel一下了), 然后写一个server.js, 里面有调用renderToString方法

然后再express的view(这个就是django里面的view)里面require一下这个server.js, 然后传入prop(初始化的map)调用.

.. code-block:: 

    function ssr(url, prop) {

        var mp = (<ExperienceDetail state={props} />);
        let markup = renderToString(mp);

        return markup
    } 




window is undefined
---------------------------------------

ssr中, Node环境是没有window的，所以必须注意去掉window或者加上type window == undefined这样的判断

特别是ComponentWillMount和ComponentDidMount这两个方法要注意



渲染了两次
---------------


ssr的时候图方便，用了<StaticRouter>

.. code-block:: 

    let markup = renderToString(
      <StaticRouter location={url} context={context}>
        <div>
                  <Route exact path="/" component={App} />
                  <Route path="/Login" component={Login} />
                  <Route path='/experience-list' component={List}></Route>
                  <Route path='/My' component={My}></Route>
                  <Route path='/Editor' component={Editor}></Route>
                  <Route path='/experience-detail' render={()=><ExperienceDetail state={props} />}></Route>
                  <Route path='/preview' component={PreView}></Route>
                  <Route path='/PublishSuccess' component={PublishSuccess}></Route>
                  <Route path='/userrole' component={UserRole}></Route>
                  <Route path='/about' component={About}></Route>
                  <Route path='/m-detail' render={()=><MDetail state={props} />}></Route>
                  <Route path='/m-my' component={MMY} ></Route>
                  <Route path='/home' component={MHOME}></Route>          
    
    
                  <Route exact path="/" component={App} />
    
                  <Route path="/Login" component={Login} />
                  <Route path='/experience-list' component={List}></Route>
                  <Route path='/My' component={My}></Route>
                  <Route path='/Editor' component={Editor}></Route>
                  <Route path='/experience-detail' render={()=><ExperienceDetail state={props} />}></Route>
                  <Route path='/preview' component={PreView}></Route>
                  <Route path='/PublishSuccess' component={PublishSuccess}></Route>
                  <Route path='/userrole' component={UserRole}></Route>
                  <Route path='/about' component={About}></Route>
    
       </div>
    
      </StaticRouter>

但是发现ssr会渲染两次，只要把StaticRouter的形式换成if else 就好了, 下面是被babel转过的，但是if else的意思是一样的


.. code-block:: 

  if (url.indexOf('/experience-detail') > -1) {
    var mp = _react2.default.createElement(_ExperienceDetail2.default, { state: props }); 
    var _markup = (0, _server.renderToString)(mp);
  }
  return _markup;

渲染速度
----------

一般性能都损失在react渲染上(别说api, api不会很慢的，相信我), 一般的解决办法只能加缓存咯~~~
  



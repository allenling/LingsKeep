react v15.6.1

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

  //server.js

  var ssr = function (url, props) {
    if (url.startsWith('/experience-detail')) {
      var mp = (<div ><ExperienceDetail state={props} /></div>);
    }else if (url.startsWith('/m-detail')) {
      var mp = (<div ><MDetail state={props} /></div>);
    }else if (url == '/') {
      var mp = (<div > <App /></div>);
    }else if (url.startsWith('/home')) {
      var mp = (<div ><MHOME /></div>);
    }
    let markup = renderToString(mp);
    return markup;
  }
  
  module.exports = ssr;



**这里注意的是需要在组件之前加一个div**

比如var mp = (<div ><ExperienceDetail state={props} /></div>);这里的div就是需要额外加的


渲染速度
----------

一般性能都损失在react渲染上(别说api, api不会很慢的，相信我), 一般的解决办法只能加缓存咯~~~

client是否会重新渲染
---------------------

react再ssr返回的文件里面会带有一个checksum的的属性

<div data-reactroot="" data-reactid="1" data-react-checksum="46739197">

通过chrome的sources来看能看到，然后再elements里面看(就是检查元素)是看不到的，被react删除掉了

checksum的作用就是react自己对比ssr过来的dom和在client的dom是否是一致的，不一致就重新渲染，一致就更新其他的dom

至于怎么通过checksum说明ssr出来的dom和client自己渲染的一致，不知道，有人说不报错就是对的，那好吧.

可以这样, 校验checksum的代码是在react-dom/lib/ReactMarkupChecksum.js:41:  canReuseMarkup: function (markup, element), 可以加一个console打印一下，

**之前网上的人说checksum被删除就是可以复用了, 但是相反, checksum被删除说明react没有复用dom, 被网上的答案骗了**

下面是追寻答案的过程:

  1. 通过chrome的performance(注意看screenshot)显示确实dom被重建了，估计是因为用户信息栏的区别~~~因为现在ssr的是没办法去获取当前用户的信息,
     要获取的话只能通过cookie传入用户的api token, 当然，更好的方法是把顶部用户信息和文章给拆成两个组件(这种本来就应该拆分的，被前端写到一起了,
     所以兼职前端写得真的很烂).
  
  2. 校验checksum的代码是在react-dom/lib/ReactMarkupChecksum.js:43(http://www.crmarsh.com/react-ssr/)

     .. code-block:: 
        // console.log是我加的打印信息

        canReuseMarkup: function (markup, element) {
          var existingChecksum = element.getAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
          existingChecksum = existingChecksum && parseInt(existingChecksum, 10);
          var markupChecksum = adler32(markup);
          var can_resue = markupChecksum === existingChecksum;
          console.log('------------client side markup: ');
          try {
            if (element.length == undefined) {
              console.log('checksum class element: ' + element.className);
            }else {
              console.log('checksum class element: ' + element[0].className);
            }   
          }   
          catch (e) {
            console.log('get checksum class element error: ' + e); 
            console.log(element);
          }   
          console.log('------------existingChecksum: ' + existingChecksum +'----markupChecksum: ' + markupChecksum);
          console.log(markup);
          console.log('------------client side can resue react: ' + can_resue);
          return can_resue;
        }

     删除checksum的代码在node_modules/react-dom/lib/ReactMount.js:471
     
     .. code-block:: 

        _mountImageIntoNode: function (markup, container, instance, shouldReuseMarkup, transaction) {
          !isValidContainer(container) ? process.env.NODE_ENV !== 'production' ? invariant(false, 'mountComponentIntoNode(...): Target container is not valid.') : _prodInvariant('41') : void 0;

          if (shouldReuseMarkup) {
            var rootElement = getReactRootElementInContainer(container);
            if (ReactMarkupChecksum.canReuseMarkup(markup, rootElement)) {
              ReactDOMComponentTree.precacheNode(instance, rootElement);
              return;
            } else {
              var checksum = rootElement.getAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
              # 这里不能复用dom的话会删除checksum!!!!!!!!!!!!!
              rootElement.removeAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);
     

  3. 根本原因，react的ComponentWillMount发起异步获取任务之后，不会等待数据加载完成，就继续render了,所以，client side第一次render出来永远是一个"空"的dom!

    3.1 可以在3.2的里面打印出信息得出是"空"的dom, react方法调用顺序:
        constructor()
        componentWillMount()
        render()
        componentDidMount()

    3.2 There’s a “gotcha,” though: An asynchronous call to fetch data will not return before the render happens. This means the component will render with empty data at least once.
        https://daveceddia.com/where-fetch-data-componentwillmount-vs-componentdidmount/

    3.3 估计只能在compoint里面先fetch data在render~~~但是这样在单页面应用的时候效果就不好，因为这样也没会卡住，这跟异步页面应用相违背了
        或者把ssr出来的数据放到html里面，比如在html里面window.__ssr__state__ = data(估计只能);

    3.4 但是这样这样的前提还是要把当前登录用户的信息拆分出来，不要放到一个component里面，因为ssr请求的时候不会带上user的token，无法拿到当前登录用户的数据，然后虽然也可以
        在cookie上带上token，但是这样~~~还是先不带吧
  
    3.5 判断是否需要复用dom在react的判断是：　var shouldReuseMarkup = containerHasReactMarkup && !prevComponent && !containerHasNonRootReactChild;
        其中containerHasReactMarkup就是<div id="root"></div>, ssr渲染之后其中必然有tag，所以client side就会去检查

    3.6 给出state都一样，初始化详情页的时候, client的markup和server side的markup老是不一样, 看了下, 在编辑器初始化的时候
        <div className='editorContainer'>
            <Editable
                placeholder="分享你的经验 ..."
                defaultValue={this.state.initialContent}
                readOnly={true}
                ref={node => { this.editor = node }}
            />
        </div>
        其中ref可能没赋值对, 跟ref没关系, 这个ref接收一个回调函数，回调函数的参数是组件的实例(这里就是Editable的实例，然后箭头函数的意思就是this.editor赋值为编辑器)                

    3.7 看来大家都是把state发送到client(https://github.com/facebook/react/issues/9681第一个回复), 可以在index.html里面<body ><script>window.__initState__ = {state}</script>...</body>
  
    3.8 出现一个现象: 第一次ssr出来到客户端的是，是可以复用的，之后就不可以了，原因是使用的编辑器slate会增加一个data-key的属性,
  　　　并且ssr的时候data-key会增加, 而client side是不会增加了, 比如
  　　　第一次的时候<editor data-key=1></editor>, 之后<editor data-key=x></editor>, 这个x值会按固定步长增加，比如7, 13, 19,...
        官方说增加了一个resetKeyGenerator函数(https://github.com/ianstormtaylor/slate/issues/53) 可以在ssr的时候key的值每次都是0



  



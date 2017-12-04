
遍历的技巧
============

例子就是slatejs的版本从0.2x升级到0.30, 这是一个不兼容的升级，其中slatejs的json结构会变化，现在把老json更新为新的json.

变为就text.ranges变为text.leaves, range变为leaf.


分析数据结构
----------------

有这样一个典型的简单的slatejs富文本编辑器(已经自定义过编辑器，所以数据跟官方例子有点差别, 但是格式都差不多)的json结构:

.. code-block:: python

   data =  {
      "kind": "value",
      "document": {
        "kind": "document",
        "data": {},
        "nodes": [
          {
            "kind": "block",
            "type": "paragraph",
            "isVoid": false,
            "data": {},
            "nodes": [
              {
                "kind": "text",
                "ranges": [
                  {
                    "kind": "range",
                    "text": "abc",
                    "marks": []
                  },
                  {
                    "kind": "range",
                    "text": "def",
                    "marks": [
                      {
                        "kind": "mark",
                        "type": "bold",
                        "data": {}
                      }
                    ]
                  },
                  {
                    "kind": "range",
                    "text": "123",
                    "marks": [
                      {
                        "kind": "mark",
                        "type": "italic",
                        "data": {}
                      }
                    ]
                  }
                ]
              }
            ]
          }
        ]
      }
    }


上面是一个最普通的文本结构, 显示出来只有一段文字: abc **def** *123* , 各个含义为:

1. document.nodes有一个元素，也就是整个document只有一段, 其中type为paragraph.

2. 这个paragraph的所有文本都包含再其nodes中: p_nodes = document.nodes[0]['nodes']

3. len(p_nodes) == 1, 这个paragraph有一个pnode, 这个pnode的type为text, 然后这个pnode里面包含三个text_node, 每一个text_node就表示文本信息，其中text_node.text字段是文本内容，而text_node.marks则是文本的样式


然后查看其他的slatejs的json结构, 比如有序列表node

1. 123
2. 456

的结构为:


.. code-block:: python

    list_items = []

    list_items[0] = {'kind': 'block', 'type': 'list-item', 'isVoid': False, 'data': {}, 'nodes': [{'kind': 'block', 'type': 'paragraph', 'isVoid': False, 'data': {}, 'nodes': [{'kind': 'text', 'range': [{'kind': 'range', 'text': '123', 'marks': []}]}]}]}
    
    list_items[1] = {'kind': 'block', 'type': 'list-item', 'isVoid': False, 'data': {}, 'nodes': [{'kind': 'block', 'type': 'paragraph', 'isVoid': False, 'data': {}, 'nodes': [{'kind': 'text', 'range': [{'kind': 'range', 'text': '456', 'marks': []}]}]}]}

    number_list_node = {
      "kind": "block",
      "type": "numbered-list",
      "isVoid": False,
      "data": {},
      "nodes": list_items
    }


上面的有序列表结构中, 每一个有序列表的type都是numbered-list, 并且nodes包含了每一个列表项目, 列表项目的type为list-item, 每一个list-item都有nodes, nodes中node都是一个paragraph,
表示了一段文本的信息, 如果list-item包含了样式:

1. 123 **abc**
2. 456 *qqq* 

其中每一个list-item['nodes']都是带样式的paragraph, 上之前提到的paragraph一样.


分析发现ranges和range都只是出现在type为text的node中, 称为text_node, 而且其他不管什么节点，总是有一个text_node节点, 因为只有text_node才包含文本内容,
并且text_node['ranges']里面就是每一段文本, type为text


所以若要把所有的ranges和range都转换成leaves和leaf, 其实只需要把所有的text_node转换就好了，所以就是如何解析出每一个节点的text_node?


遍历思路
-------------

要求: in-place替换, 不然各种赋值比较麻烦, 对一个text_node, 替换函数为


.. code-block:: python

    def parse_text_node(text_node):
        if 'ranges' not in text_node:
            return
        r = text_node.pop('ranges')
        for i in r:
            if 'kind' in i and i['kind'] == 'range':
                i['kind'] = 'leaf'
        text_node['leaves'] = r
        return

然后我们遍历data['document']['nodes'], 每一个类型的node都有自己的结构，但是最终可能有一个text_node, 我们可以根据每一个node的特点，对应解析


1. 为每一个类型的node写一个parse函数，循环data['document']['nodes']中每一个节点，判断节点类型，调用对应的解析函数.

.. code-block:: python


    def parse_numbered_list(data):
        for list_item in data['nodes']:
            for paragraph in list_item['nodes']:
                for text_node in paragraph['nodes']:
            	    parse_text_node(list_item)
        return

    for node in data['document']['nodes']:
        if node['type'] == 'numbered-list':
            parse_numbered_list(node)


这样的话, 比较清晰，但是也有麻烦的地方，每一个类型都需要写一个解析函数, 加入新类型的话就不好弄了.


2. 根据这样一个特点，不管什么节点，最终都会包含一个text_node, 我们只需要对每一个节点，找出最深层级的text_node就好了.


2.1 递归
++++++++++

不喜欢


2.2 调用栈式
++++++++++++++


考虑一下二叉树的前序，中序，后序遍历，也是一个调用栈式的是写法:

.. code-block:: python

    def pre_order(root):
        '''
        前序遍历, 根节点->左子节点->右子节点
        '''
        tmp = [root]
        res = []
        while tmp:
            node = tmp.pop(0)
            if not node:
                continue
            tmp.insert(0, node.right)
            tmp.insert(0, node.left)
            res.append(node.value)
        return res

遍历节点都先pop, 然后解析, 对于一个列表，每一个元素都有内嵌元素，遍历的时候可以先pop出元素, 然后把需要再遍历的子元素添加到遍历列表中, 比如slatejs结构里面
每一个node都有nodes这个key, 然后把node['nodes']的元素insert到列表的前面


.. code-block:: python

    def stack_call(node_list):
        while node_list:
            node = node_list.pop(0):
            for subnode in node['nodes']:
                if subnode['type'] != 'text':
                    node_list.insert(0, subnode)
                else:
                    parse_text_node(subnode)

**但是这样有个问题，因为pop之后整个node_list的结构就变了.**

根据python对象引用的特点，可以把subnode插入到列表头, 这样就算node_list.pop(0)也不会影响到node_list的结构


.. code-block:: python

    def stack_call(node_list):
        for node in node_list:
            for subnode in node['nodes']:
                if subnode['type'] != 'text':
                    node_list.insert(0, subnode)
                else:
                    parse_text_node(subnode)


但是这样不行的, for的话不会从新遍历改变后的node_list, 用while也是死路(这里还有其他奇奇怪怪的死路，但是node_list上用pop总归不行)

既然是弹调用栈的形式, pop还得继续用, 如果继续在node_list上用pop, 那会改变node_list的结构,
但是我们可以在每次遍历ndoe['nodes']的时候，用一个new_node_list, 在new_node_list上pop, 每次把subnode插入到new_node_list上, 这样就可以了


.. code-block:: python

    def trans_ranges_to_leaves(text_node):
        if 'ranges' not in text_node:
            return
        r = text_node.pop('ranges')
        for i in r:
            if 'kind' in i and i['kind'] == 'range':
                i['kind'] = 'leaf'
        text_node['leaves'] = r
        return
    
    
    def extract_text_plain(text_node, key='leaves'):
        # 这里只是解析其中文本而已
        data = []
        for text in text_node[key]:
            data.append(text['text'])
        return ''.join(data)
    
    
    def extract_document(data):
        plain_text = []
        ns = data['document']['nodes']
        for n in ns:
            # 并不在ns上pop, 需要保持结构
            # 用一个iter_nodes来对node进行调用栈式的遍历
            # 这里注意的是用一个列表来包装一个遍历的节点，做个调用栈, 这个很关键
            # 弹栈式的时候这个小技巧很好用
            iter_nodes = [n]
            node_plain_text = []
            while iter_nodes:
                # 在这里pop不会影响ns的结构，并且由于python对象引用的特点
                # 操作的是同一个对象，可以in-place替换
                sub_iter = iter_nodes.pop(0)
                if 'nodes' in sub_iter:
                    sub_nodes_list = []
                    for sub_nodes_node in sub_iter['nodes']:
                        if 'kind' in sub_nodes_node and sub_nodes_node['kind'] != 'text':
                            # insert和append注意下，不然输出的plain_text是倒序的
                            sub_nodes_list.append(sub_nodes_node)
                        else:
                            trans_ranges_to_leaves(sub_nodes_node)
                            # insert和append注意下，不然输出的plain_text是倒序的
                            node_plain_text.append(extract_text_plain(sub_nodes_node))
                    if sub_nodes_list:
                        # sub_nodes_list + iter_nodes 还是 iter_nodes + sub_nodes_list 注意下，不然输出的plain_text是倒序的
                        iter_nodes = sub_nodes_list + iter_nodes
            node_plain_text = ''.join(node_plain_text)
            plain_text.append(node_plain_text)
        return plain_text
    
    
    def main():
        j_str = '一个slatejs的json结构字符串'
        jdata = json.loads(j_str)
        for i in jdata['document']['nodes']:
            print(i)
        print('---------------')
        plain_text = extract_document(jdata)
        for i in jdata['document']['nodes']:
            print(i)
        print('+++++++++++++++')
        for j in plain_text:
            print(j)


一个比较复杂的slatejs结构, 带ranges和range:

.. code-block::

    {"kind":"value","document":{"kind":"document","data":{},"nodes":[{"kind":"block","type":"numbered-list","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"list-item","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"sdfa","marks":[]},{"kind":"leaf","text":"sdsf ","marks":[{"kind":"mark","type":"bold","data":{}}]}]},{"kind":"inline","type":"link","isVoid":false,"data":{"href":"http://32t4ry"},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"dfjtrjytt","marks":[{"kind":"mark","type":"bold","data":{}}]}]}]},{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]}]}]},{"kind":"block","type":"list-item","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"dsfdhg32","marks":[]}]}]}]}]},{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"sadqresfewgfrewg","marks":[]}]}]},{"kind":"block","type":"bulleted-list","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"list-item","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"safsdhfdjhgfj","marks":[]}]}]}]},{"kind":"block","type":"list-item","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"qewqfrewg","marks":[]}]}]}]},{"kind":"block","type":"list-item","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"sfdsg","marks":[]}]}]}]}]},{"kind":"block","type":"heading-two","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"rtjutrkjyk","marks":[]}]}]},{"kind":"block","type":"block-quote","isVoid":false,"data":{},"nodes":[{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"yj5yej","marks":[]}]}]}]},{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]}]},{"kind":"block","type":"divider","isVoid":true,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":" ","marks":[]}]}]},{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]},{"kind":"inline","type":"link","isVoid":false,"data":{"href":"http://54eytruj"},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"hgfrjhgkjhl,","marks":[]}]}]},{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]}]},{"kind":"block","type":"paragraph","isVoid":false,"data":{},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]},{"kind":"inline","type":"link","isVoid":false,"data":{"href":"http://4654u"},"nodes":[{"kind":"text","ranges":[{"kind":"leaf","text":"嘿嘿嘿","marks":[]}]}]},{"kind":"text","ranges":[{"kind":"leaf","text":"","marks":[]}]}]}]}}


post save foreignkey
======================

V2.3中, 要保存post过来的foreignkey的id值, 必须在serializer中\ **显式**\ 写上foreignkey的字段名.


parser
===========

v3.6.3

rest framework默认的request的parser在rest_framework.parser.JSONParser, 然后用的是自带的json来loads


.. code-block:: python

  class JSONParser(BaseParser):
      media_type = 'application/json'
      renderer_class = renderers.JSONRenderer
  
      def parse(self, stream, media_type=None, parser_context=None):
          """
          Parses the incoming bytestream as JSON and returns the resulting data.
          """
          parser_context = parser_context or {}
          encoding = parser_context.get('encoding', settings.DEFAULT_CHARSET)
  
          try:
              data = stream.read().decode(encoding)
              # 这里的json是默认的json
              return json.loads(data)
          except ValueError as exc:
              raise ParseError('JSON parse error - %s' % six.text_type(exc))
  

根据http://artem.krylysov.com/blog/2015/09/29/benchmark-python-json-libraries/ 的测试，ujson更快一点.

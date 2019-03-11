
从stdin读取命令执行并且保存输出
===================================



.. code-block:: python

    '''
    
    #!/bin/bash
    
    exec 3>&1 4>&2
    
    read cmd
    while [ "$cmd" != "exit" ];
    do
        output=($($cmd 2>&1))
        echo $output_len
        echo $output
        read cmd
    done
    
    '''



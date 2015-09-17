获取文件创建时间和修改时间
===========================

Python中可以用os.path.getctime和os.path.getmtime来获取文件创建时间和修改时间.

其中获取创建时间有点区别.

getctime是调用系统调用stat, 也就是相当于调用os.stat获取返回的st_ctime

但是,各个平台下不太一样,getctime在Windows下是创建时间,而*nix下,有些系统,比如linux则是inode信息(meta data)最后被修改的时间,chmod, chown等操作都会修改inode的修改时间.

**man state(2)** 中对于st_ctime的描述: `The field st_ctime is changed by writing or by setting inode information (i.e., owner, group, link count, mode, etc.)`

stackoverflower: http://stackoverflow.com/questions/237079/how-to-get-file-creation-modification-date-times-in-python


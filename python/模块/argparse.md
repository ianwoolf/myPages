+++
date = "2015-12-21T16:30:09+08:00"
draft = true
title = "argparse 命令行参数解析模块"

+++

python发展的时间很长,功能很全.是devops的得力工具.
我的python太弱了,一直想写python的东西,做个积累.一直没有做,从argparse开始吧.

    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument('--list', action='append', dest='list', help='append')
    parser.add_argument('--false', action='store_false', dest='false',
        help='store_false')
    parser.add_argument('--true', action='store_true', dest='true',
        help='stroe_true')
    parser.add_argument('--foo', help='fault store action')

    print parser.parse_args('--foo 2'.split())
    #Namespace(false=True, foo='2', list=None, true=False)
    print parser.parse_args('--foo x'.split())
    #Namespace(false=True, foo='x', list=None, true=False)
    print parser.parse_args('--false'.split())
    #Namespace(false=False, foo=None, list=None, true=False)
    print parser.parse_args('--true'.split())
    #Namespace(false=True, foo=None, list=None, true=True)
    print parser.parse_args('--list 1, --list 2'.split())
    #Namespace(false=True, foo=None, list=['1,', '2'], true=False)

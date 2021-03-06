37 Miles
===============

setuptools, easy_install概览和第三包的安装位置
-----------------------------------------------
setuptools基于标准库distutils，为python提供了一种简便的安装第三方
程序模块的方式, easy_install命令由setuptools 包提供。

源码::

    jaime@westeros:~/Downloads/setuptools-0.6c11/setuptools$ ls
    archive_util.py  command     dist.py       gui.exe      package_index.py tests
    cli.exe          depends.py  extension.py  __init__.py  sandbox.py

以Flask为例::

    jaime@westeros:~/Downloads/Flask-0.7.2$ ls
    artwork  build    dist  examples  Flask.egg-info  Makefile     PKG-INFO setup.cfg  tests
    AUTHORS  CHANGES  docs  flask     LICENSE         MANIFEST.in  README setup.py

setup.py内容::

    from setuptools import Command, setup
    ...

    setup(
        name='Flask',
        version='0.7.2',
        url='http://github.com/mitsuhiko/flask/',
        license='BSD',
        author='Armin Ronacher',
        author_email='armin.ronacher@active-4.com',
        description='A microframework based on Werkzeug, Jinja2 '
                    'and good intentions',
        long_description=__doc__,
        packages=['flask'],
        zip_safe=False,
        platforms='any',
        install_requires=[
            'Werkzeug>=0.6.1',
            'Jinja2>=2.4'
        ],
        classifiers=[
        ...
        ],
        cmdclass={'audit': run_audit},
        test_suite='__main__.run_tests'
    )

可见setup函数来自于setuptools模块，setuptools/__init__.py 代码::

    """Extensions to the 'distutils' for large or complex distributions"""
    from setuptools.extension import Extension, Library

    # import Distribution, 实际上在setuptools/dist.py里已经做了一些monkey-patch的替换工作了
    from setuptools.dist import Distribution, Feature, _get_unpatched
    import distutils.core, setuptools.command
    from setuptools.depends import Require
    from distutils.core import Command as _Command
    from distutils.util import convert_path
    import os.path

    __version__ = '0.6c11'
    __all__ = [
        'setup', 'Distribution', 'Feature', 'Command', 'Extension', 'Require',
        'find_packages'
    ]

    bootstrap_install_from = None

    def find_packages(where='.', exclude=()):
        """Return a list all Python packages found within directory 'where'
        ...

    # setup函数实际上来自于distutils标准库
    setup = distutils.core.setup 
        
    _Command = _get_unpatched(_Command)

    class Command(_Command):
        ...

    import distutils.core
    distutils.core.Command = Command    # we can't patch distutils.cmd, alas

    def findall(dir = os.curdir):
        """Find all files under 'dir' and return the list of full filenames
        ...

    import distutils.filelist
    distutils.filelist.findall = findall    # fix findall bug in distutils.


distutils/core.py, setup是 distutils 模块的总入口，它创建一个 Distribution
实例，解析配置参数，然后运行具体的命令::

    def setup (**attrs):
        """The gateway to the Distutils: do everything your setup script needs
        to do, in a highly flexible and user-driven way.  Briefly: create a
        Distribution instance; find and parse config files; parse the command
        line; run each Distutils command found there, customized by the options
        supplied to 'setup()' (as keyword arguments), in config files, and on
        the command line.

        The Distribution instance might be an instance of a class supplied via
        the 'distclass' keyword argument to 'setup'; if no such class is
        supplied, then the Distribution class (in dist.py) is instantiated.
        All other arguments to 'setup' (except for 'cmdclass') are used to set
        attributes of the Distribution instance.
        ...

        """

        global _setup_stop_after, _setup_distribution

        # Determine the distribution class -- either caller-supplied or
        # our Distribution (see below).
        klass = attrs.get('distclass')
        if klass:
            del attrs['distclass']
        else:
            # 默认情况下，走这里，注意，这里的Distribution已经是被setuptools打过补丁的了, 
            # 代码后续奉上
            klass = Distribution
        ...

        # Create the Distribution instance, using the remaining arguments
        # (ie. everything except distclass) to initialize it
        try:
            _setup_distribution = dist = klass(attrs)
        except DistutilsSetupError, msg:
            if 'name' in attrs:
                raise SystemExit, "error in %s setup command: %s" % \
                      (attrs['name'], msg)
            else:
                raise SystemExit, "error in setup command: %s" % msg
        ...

        # Find and parse the config file(s): they will override options from
        # the setup script, but be overridden by the command line.
        dist.parse_config_files()
        ...

        # Parse the command line; any command-line errors are the end user's
        # fault, so turn them into SystemExit to suppress tracebacks.
        try:
            ok = dist.parse_command_line()
        except DistutilsArgError, msg:
            raise SystemExit, gen_usage(dist.script_name) + "\nerror: %s" % msg
        ...

        # And finally, run all the commands found on the command line.
        if ok:
            try:
                dist.run_commands()
            except KeyboardInterrupt:
                raise SystemExit, "interrupted"
            ...

        return dist

setuptools/dist.py::

    from distutils.core import Distribution as _Distribution
    ...

    _Distribution = _get_unpatched(_Distribution)

    class Distribution(_Distribution):
        """Distribution with support for features, tests, and package data

        This is an enhanced version of 'distutils.dist.Distribution' that
        effectively adds the following new optional keyword arguments to 'setup()':
        ...

    # 替换 Distribution 的地方
    # Install it throughout the distutils
    for module in distutils.dist, distutils.core, distutils.cmd:
        module.Distribution = Distribution

既然dist已经是setuptools.dist.Distribution的一个实例，跟踪代码的时候需要分清
函数到底是在什么地方定义，distutils还是setuptools.

dist.run_commands 继承自distutils/dist.py::

   def run_commands(self):
        """Run each command that was seen on the setup script command line.
        Uses the list of commands found and cache of command objects
        created by 'get_command_obj()'.
        """
        for cmd in self.commands:
            self.run_command(cmd)

    def run_command(self, command):
        # Already been here, done that? then return silently.
        if self.have_run.get(command):
            return

        log.info("running %s", command)
        cmd_obj = self.get_command_obj(command)
        cmd_obj.ensure_finalized()
        cmd_obj.run()
        self.have_run[command] = 1

总之，对于python setup.py install来说，这个cmd_obj就是setuptools/command/install.py了，
设个trace看看::

    jaime@westeros:/home/ideer/Downloads/Flask-0.7.2$ diff -u /usr/local/lib/python2.6/distutils/dist.py.orig /usr/local/lib/python2.6/distutils/dist.py
    --- /usr/local/lib/python2.6/distutils/dist.py.orig	2011-10-10 22:27:53.658046001 +0800
    +++ /usr/local/lib/python2.6/distutils/dist.py	2011-10-10 22:28:17.350163485 +0800
    @@ -989,6 +989,7 @@
             if self.have_run.get(command):
                 return
     
    +        import pdb; pdb.set_trace()
             log.info("running %s", command)
             cmd_obj = self.get_command_obj(command)
             cmd_obj.ensure_finalized()

    jaime@westeros:/home/ideer/Downloads/Flask-0.7.2$ python setup.py install
    > /usr/local/lib/python2.6/distutils/dist.py(993)run_command()
    -> log.info("running %s", command)
    (Pdb) n
    running install
    > /usr/local/lib/python2.6/distutils/dist.py(994)run_command()
    -> cmd_obj = self.get_command_obj(command)
    (Pdb) 
    (Pdb) cmd_obj.__class__
    <class setuptools.command.install.install at 0xb7440dac>
    (Pdb) 


setuptools/command/install.py::

        def run(self):
            ...
            if caller_module != 'distutils.dist' or caller_name!='run_commands':
                # We weren't called from the command line or setup(), so we
                # should run in backward-compatibility mode to support bdist_*
                # commands.
                _install.run(self)
            else:
                self.do_egg_install()

        def do_egg_install(self):

            easy_install = self.distribution.get_command_class('easy_install')

            cmd = easy_install(
                self.distribution, args="x", root=self.root, record=self.record,
            )
            # 注意这里，将决定安装位置的地方，后续详述
            cmd.ensure_finalized()  # finalize before bdist_egg munges install cmd
            cmd.always_copy_from = '.'  # make sure local-dir eggs get installed

            # pick up setup-dir .egg files only: no .egg-info
            cmd.package_index.scan(glob.glob('*.egg'))

            self.run_command('bdist_egg')
            args = [self.distribution.get_command_obj('bdist_egg').egg_output]

            if setuptools.bootstrap_install_from:
                # Bootstrap self-installation of setuptools
                args.insert(0, setuptools.bootstrap_install_from)

            # 跳转到setuptools/command/easy_install.py的run
            cmd.args = args
            cmd.run()
            setuptools.bootstrap_install_from = None

bdist_egg 命令可谓做了整个安装过程的大部分工作，准备egg文件，编译，直到生成egg压缩包
为止，代码在setuptools/command/bdist_egg.py，编译c扩展的命令 build_clib 在distutils中完成，
distutils/command/build_clib.py。

有了egg文件之后，setuptools/command/easy_install.py::

    def run(self):
        if self.verbose!=self.distribution.verbose:
            log.set_verbosity(self.verbose)
        try:
            for spec in self.args:
                self.easy_install(spec, not self.no_deps)
            ...
        finally:
            log.set_verbosity(self.distribution.verbose)

    def easy_install(self, spec, deps=False):
        tmpdir = tempfile.mkdtemp(prefix="easy_install-")
        download = None
        if not self.editable: self.install_site_py()

        # 根据spec的不同，有三种情况需要处理
        try:
            if not isinstance(spec,Requirement):
                if URL_SCHEME(spec):
                    # 1. 需要下载的包
                    # It's a url, download it to tmpdir and process
                    self.not_editable(spec)
                    download = self.package_index.download(spec, tmpdir)
                    return self.install_item(None, download, tmpdir, deps, True)

                elif os.path.exists(spec):
                    # 2. 本地的包
                    # Existing file or directory, just process it directly
                    self.not_editable(spec)
                    return self.install_item(None, spec, tmpdir, deps, True)
                else:
                    spec = parse_requirement_arg(spec)

            # 3. 查找某个依赖的包
            # spec 是 Requirement
            self.check_editable(spec)
            dist = self.package_index.fetch_distribution(
                spec, tmpdir, self.upgrade, self.editable, not self.always_copy,
                self.local_index
            )
            if dist is None:
                msg = "Could not find suitable distribution for %r" % spec
                if self.always_copy:
                    msg+=" (--always-copy skips system and development eggs)"
                raise DistutilsError(msg)
            elif dist.precedence==DEVELOP_DIST:
                # .egg-info dists don't need installing, just process deps
                self.process_distribution(spec, dist, deps, "Using")
                return dist
            else:
                return self.install_item(spec, dist.location, tmpdir, deps)

        finally:
            if os.path.exists(tmpdir):
                rmtree(tmpdir)

    def install_item(self, spec, download, tmpdir, deps, install_needed=False):

        # Installation is also needed if file in tmpdir or is not an egg
        install_needed = install_needed or self.always_copy
        install_needed = install_needed or os.path.dirname(download) == tmpdir
        install_needed = install_needed or not download.endswith('.egg')
        install_needed = install_needed or (
            self.always_copy_from is not None and
            os.path.dirname(normalize_path(download)) ==
            normalize_path(self.always_copy_from)
        )

        if spec and not install_needed:
            # at this point, we know it's a local .egg, we just don't know if
            # it's already installed.
            for dist in self.local_index[spec.project_name]:
                if dist.location==download:
                    break
            else:
                install_needed = True   # it's not in the local index

        # 注意这个 marker
        log.info("Processing %s", os.path.basename(download))

        if install_needed:
            dists = self.install_eggs(spec, download, tmpdir)
            # install_eggs返回成功安装后的包信息, 参见self.egg_distribution 
            # 一般是只有一个元素的list，但可能一次安装了多个包，
            ＃所以下面的for，遍历处理所有成功安装的包依赖关系
            for dist in dists:
                self.process_distribution(spec, dist, deps)
        else:
            dists = [self.check_conflicts(self.egg_distribution(download))]
            self.process_distribution(spec, dists[0], deps, "Using")

        if spec is not None:
            for dist in dists:
                if dist in spec:
                    return dist

    def process_distribution(self, requirement, dist, deps=True, *info):
        # 处理后续安装事宜
        # requirement是spec，dist是成功安装包的信息, deps是否处理依赖关系的标志
        self.update_pth(dist)
        self.package_index.add(dist)
        self.local_index.add(dist)
        self.install_egg_scripts(dist)
        self.installed_projects[dist.key] = dist
        log.info(self.installation_report(requirement, dist, *info))
        if dist.has_metadata('dependency_links.txt'):
            self.package_index.add_find_links(
                dist.get_metadata_lines('dependency_links.txt')
            )
        if not deps and not self.always_copy:
            return # 没有依赖关系，done

        elif requirement is not None and dist.key != requirement.key:
            log.warn("Skipping dependencies for %s", dist)
            return  # XXX this is not the distribution we were looking for
        elif requirement is None or dist not in requirement:
            # if we wound up with a different version, resolve what we've got
            distreq = dist.as_requirement()
            requirement = requirement or distreq
            requirement = Requirement(
                distreq.project_name, distreq.specs, requirement.extras
            )

        # 注意这个marker
        log.info("Processing dependencies for %s", requirement)
        try:
            # 获得依赖关系的安装信息
            distros = WorkingSet([]).resolve(
                [requirement], self.local_index, self.easy_install
            )
        except DistributionNotFound, e:
            raise DistutilsError(
                "Could not find required distribution %s" % e.args
            )
        except VersionConflict, e:
            raise DistutilsError(
                "Installed distribution %s conflicts with requirement %s"
                % e.args
            )
        if self.always_copy or self.always_copy_from:
            # Force all the relevant distros to be copied or activated
            for dist in distros:
                if dist.key not in self.installed_projects:
                    # 嵌套依赖关系安装
                    # 又回到easy_install，因为依赖的包可能也依赖别的包
                    # 可能也需要从pypi下载
                    self.easy_install(dist.as_requirement())
        # Marker
        log.info("Finished processing dependencies for %s", requirement)

把egg文件解压到系统目录的是install_eggs函数::

    def install_eggs(self, spec, dist_filename, tmpdir):
        # .egg dirs or files are already built, so just return them
        if dist_filename.lower().endswith('.egg'):
            # 调用install_egg安装
            return [self.install_egg(dist_filename, tmpdir)]
        elif dist_filename.lower().endswith('.exe'):
            return [self.install_exe(dist_filename, tmpdir)]

        # Anything else, try to extract and build
        # 下载的.tar.gz包在这里处理
        setup_base = tmpdir
        if os.path.isfile(dist_filename) and not dist_filename.endswith('.py'):
            unpack_archive(dist_filename, tmpdir, self.unpack_progress)
        elif os.path.isdir(dist_filename):
            setup_base = os.path.abspath(dist_filename)
        ...

        # Find the setup.py file
        setup_script = os.path.join(setup_base, 'setup.py')
        ...

        # Now run it, and return the result
        if self.editable:
            log.info(self.report_editable(spec, setup_script))
            return []
        else:
            return self.build_and_install(setup_script, setup_base)

    def install_egg(self, egg_path, tmpdir):
        # self.install_dir 这里！
        destination = os.path.join(self.install_dir,os.path.basename(egg_path))
        destination = os.path.abspath(destination)
        if not self.dry_run:
            ensure_directory(destination)

        dist = self.egg_distribution(egg_path)
        self.check_conflicts(dist)
        if not samefile(egg_path, destination):
            if os.path.isdir(destination) and not os.path.islink(destination):
                dir_util.remove_tree(destination, dry_run=self.dry_run)
            elif os.path.exists(destination):
                self.execute(os.unlink,(destination,),"Removing "+destination)
            uncache_zipdir(destination)
            if os.path.isdir(egg_path):
                if egg_path.startswith(tmpdir):
                    f,m = shutil.move, "Moving"
                else:
                    f,m = shutil.copytree, "Copying"
            elif self.should_unzip(dist):
                self.mkpath(destination)
                # egg包调用unpack_and_compile解压，编译
                f,m = self.unpack_and_compile, "Extracting"
            elif egg_path.startswith(tmpdir):
                f,m = shutil.move, "Moving"
            else:
                f,m = shutil.copy2, "Copying"

            self.execute(f, (egg_path, destination),
                (m+" %s to %s") %
                (os.path.basename(egg_path),os.path.dirname(destination)))

        self.add_output(destination)
        return self.egg_distribution(destination)

终于找到self.install_dir, 才是决定安装位置的所在。那么它又从哪里来的呢？

setuptools/command/easy_install.py finalize_options函数，在上文的cmd.ensure_finalized(继承自distutils.cmd.Command)中被调用::

    def finalize_options(self):
        # 如果setup.py install命令附带了--prefix 参数，则在 _expand 函数中处理
        # 该文件中的 INSTALL_SCHEMES 即作此用，与distutils install.py中的不同
        self._expand('install_dir','script_dir','build_directory','site_dirs')
        # If a non-default installation directory was specified, default the
        # script directory to match it.
        if self.script_dir is None:
            self.script_dir = self.install_dir
        
        # 默认没有指定
        # Let install_dir get set by install_lib command, which in turn
        # gets its info from the install command, and takes into account
        # --prefix and --home and all that other crud.
        self.set_undefined_options('install_lib',
            ('install_dir','install_dir')
        )
        # 意思是将install_lib命令的install_dir属性赋予本命令的install_dir
 
distutils/command/install_lib.py::

    def finalize_options (self):

        # Get all the information we need to install pure Python modules
        # from the umbrella 'install' command -- build (source) directory,
        # install (target) directory, and whether to compile .py files.
        self.set_undefined_options('install',
                                   ('build_lib', 'build_dir'),
                                   ('install_lib', 'install_dir'),
                                   ('force', 'force'),
                                   ('compile', 'compile'),
                                   ('optimize', 'optimize'),
                                   ('skip_build', 'skip_build'),
                                  )

       # 意思是将install命令的install_lib属性赋予本命令的install_dir

令人费解的finalize_options！大意是每个命令都需要有这么一个机会，在最后时刻
设置一些参数，easy_install从install_lib中，后者从install中得到了位置信息。

目标install命令的 install_lib。       

setuptools.command.install命令的finalize_options函数，调用了distutils.command.install.finalize_options，

distutils/command/install.py::

    INSTALL_SCHEMES = {
        'unix_prefix': {
            'purelib': '$base/lib/python$py_version_short/site-packages',
            'platlib': '$platbase/lib/python$py_version_short/site-packages',
            'headers': '$base/include/python$py_version_short/$dist_name',
            'scripts': '$base/bin',
            'data'   : '$base',
            },
        'unix_home': {
            'purelib': '$base/lib/python',
            'platlib': '$base/lib/python',
            'headers': '$base/include/python/$dist_name',
            'scripts': '$base/bin',
            'data'   : '$base',
            },
            ...
        }

    def finalize_options (self):


        # This method (and its pliant slaves, like 'finalize_unix()',
        # 'finalize_other()', and 'select_scheme()') is where the default
        # installation directories for modules, extension modules, and
        # anything else we care to install from a Python module
        # distribution.  Thus, this code makes a pretty important policy
        # statement about how third-party stuff is added to a Python
        # installation!  Note that the actual work of installation is done
        # by the relatively simple 'install_*' commands; they just take
        # their orders from the installation directory options determined
        # here.

        ...

        # Now the interesting logic -- so interesting that we farm it out
        # to other methods.  The goal of these methods is to set the final
        # values for the install_{lib,scripts,data,...}  options, using as
        # input a heady brew of prefix, exec_prefix, home, install_base,
        # install_platbase, user-supplied versions of
        # install_{purelib,platlib,lib,scripts,data,...}, and the
        # INSTALL_SCHEME dictionary above.  Phew!

        self.dump_dirs("pre-finalize_{unix,other}")

        if os.name == 'posix':
            # 神秘的选中 unix_prefix scheme
            self.finalize_unix()
        else:
            self.finalize_other()

        self.dump_dirs("post-finalize_{unix,other}()")
        ...

        # Pick the actual directory to install all modules to: either
        # install_purelib or install_platlib, depending on whether this
        # module distribution is pure or not.  Of course, if the user
        # already specified install_lib, use their selection.
        if self.install_lib is None:
            if self.distribution.ext_modules: # has extensions: non-pure
                self.install_lib = self.install_platlib
            else:

                # INSTALL_SCHEMES 里为什么是purelib？原因在这里
                # 将purelib路径赋值给install_lib, done
                self.install_lib = self.install_purelib 

        ...

    def finalize_unix (self):

        if self.install_base is not None or self.install_platbase is not None:
            ...
            return

        if self.user:
            ...
            self.select_scheme("unix_user")
        elif self.home is not None:
            self.install_base = self.install_platbase = self.home
            self.select_scheme("unix_home")
        else:
            if self.prefix is None:
                if self.exec_prefix is not None:
                    raise DistutilsOptionError, \
                          "must not supply exec-prefix without prefix"

                self.prefix = os.path.normpath(sys.prefix)
                self.exec_prefix = os.path.normpath(sys.exec_prefix)

            else:
                if self.exec_prefix is None:
                    self.exec_prefix = self.prefix

            self.install_base = self.prefix
            self.install_platbase = self.exec_prefix
            self.select_scheme("unix_prefix")

finalize_unix -> unix_prefix scheme -> purelib -> install_lib -> install_dir

太复杂了, 正如其注释所言，各种和安装目录有关的情况都在此处理，若要细究可在此设置断点跟踪。

一些debug信息::

    pre-finalize_{unix,other}:
      prefix: None
      exec_prefix: None
      home: None
      user: 0
      install_base: None
      install_platbase: None
      root: None
      install_purelib: None
      install_platlib: None
      install_lib: None
      install_headers: None
      install_scripts: None
      install_data: None
      ...
    post-finalize_{unix,other}():
      prefix: /usr/local
      exec_prefix: /usr/local
      home: None
      user: 0
      install_base: /usr/local
      install_platbase: /usr/local
      root: None
      install_purelib: $base/lib/python$py_version_short/site-packages
      install_platlib: $platbase/lib/python$py_version_short/site-packages
      install_lib: None
      install_headers: $base/include/python$py_version_short/$dist_name
      install_scripts: $base/bin
      install_data: $base
      ...
    after prepending root:
      prefix: /usr/local
      exec_prefix: /usr/local
      home: None
      user: 0
      install_base: /usr/local
      install_platbase: /usr/local
      root: None
      install_purelib: /usr/local/lib/python2.6/site-packages
      install_platlib: /usr/local/lib/python2.6/site-packages
      install_lib: /usr/local/lib/python2.6/site-packages/
      install_headers: /usr/local/include/python2.6/Flask
      install_scripts: /usr/local/bin
      install_data: /usr/local
      ...

实际输出::

    jaime@westeros:/home/ideer/Downloads/Flask-0.7.2$ sudo python setup.py install
    running install
    running bdist_egg
    # 获得EGG-INFO
    running egg_info
    writing requirements to Flask.egg-info/requires.txt
    writing Flask.egg-info/PKG-INFO
    writing top-level names to Flask.egg-info/top_level.txt
    writing dependency_links to Flask.egg-info/dependency_links.txt
    reading manifest file 'Flask.egg-info/SOURCES.txt'
    reading manifest template 'MANIFEST.in'
    ...
    writing manifest file 'Flask.egg-info/SOURCES.txt'
    installing library code to build/bdist.linux-i686/egg
    running install_lib
    running build_py
    # 建立build目录，build/lib临时存放，最终目录为build/dist目录
    creating build/lib
    creating build/lib/flask
    copying flask/views.py -> build/lib/flask
    copying flask/helpers.py -> build/lib/flask
    ...
    # 建立 egg 包根目录
    creating build/bdist.linux-i686/egg
    creating build/bdist.linux-i686/egg/flask
    # 复制纯py文件到egg目录
    copying build/lib/flask/views.py -> build/bdist.linux-i686/egg/flask
    copying build/lib/flask/helpers.py -> build/bdist.linux-i686/egg/flask
    ...
    # 编译
    byte-compiling build/bdist.linux-i686/egg/flask/views.py to views.pyc
    byte-compiling build/bdist.linux-i686/egg/flask/helpers.py to helpers.pyc
    ...
    # 将准备好的EGG-INFO复制到egg目录
    creating build/bdist.linux-i686/egg/EGG-INFO
    copying Flask.egg-info/PKG-INFO -> build/bdist.linux-i686/egg/EGG-INFO
    copying Flask.egg-info/SOURCES.txt -> build/bdist.linux-i686/egg/EGG-INFO
    ...
    # 将egg根目录压缩为egg文件，存放到dist目录
    creating 'dist/Flask-0.7.2-py2.6.egg' and adding 'build/bdist.linux-i686/egg' to it
    removing 'build/bdist.linux-i686/egg' (and everything under it)
    # 开始安装 Flask egg
    Processing Flask-0.7.2-py2.6.egg
    creating /usr/local/lib/python2.6/site-packages/Flask-0.7.2-py2.6.egg
    # 加压本地egg
    Extracting Flask-0.7.2-py2.6.egg to /usr/local/lib/python2.6/site-packages
    Adding Flask 0.7.2 to easy-install.pth file

    Installed /usr/local/lib/python2.6/site-packages/Flask-0.7.2-py2.6.egg
    # process_distribution 标志输出
    Processing dependencies for Flask==0.7.2
    Searching for Jinja2>=2.4
    Reading http://pypi.python.org/simple/Jinja2/
    Reading http://jinja.pocoo.org/
    Best match: Jinja2 2.6
    Downloading http://pypi.python.org/packages/source/J/Jinja2/Jinja2-2.6.tar.gz#md5=1c49a8825c993bfdcf55bb36897d28a2
    # install_item 标志输出，开始安装jinja
    Processing Jinja2-2.6.tar.gz
    Running Jinja2-2.6/setup.py -q bdist_egg --dist-dir /tmp/easy_install-1wzF7P/Jinja2-2.6/egg-dist-tmp-g9b8N8
    ...
    Adding Jinja2 2.6 to easy-install.pth file

    Installed /usr/local/lib/python2.6/site-packages/Jinja2-2.6-py2.6.egg
    Searching for Werkzeug>=0.6.1
    Reading http://pypi.python.org/simple/Werkzeug/
    Reading http://werkzeug.pocoo.org/
    Reading http://trac.pocoo.org/repos/werkzeug/trunk
    Best match: Werkzeug 0.8.1
    Downloading http://pypi.python.org/packages/source/W/Werkzeug/Werkzeug-0.8.1.tar.gz#md5=20f3a65710d64f9f455111ed71e3da66
    # install_item 标志输出，开始安装werkzeug
    Processing Werkzeug-0.8.1.tar.gz
    Running Werkzeug-0.8.1/setup.py -q bdist_egg --dist-dir /tmp/easy_install-SUNJMn/Werkzeug-0.8.1/egg-dist-tmp-uZjjba
    ...
    Adding Werkzeug 0.8.1 to easy-install.pth file

    Installed /usr/local/lib/python2.6/site-packages/Werkzeug-0.8.1-py2.6.egg
    # process_distribution 所有Flask依赖关系处理完毕
    Finished processing dependencies for Flask==0.7.2
    jaime@westeros:/home/ideer/Downloads/Flask-0.7.2$ 


其实不管安装工具多么复杂，最主要的有两点：

#. 如果是纯py代码，那么复制到python路径就行了，比如site-packages

#. 如果是python c扩展，则需要找到python头文件，其他依赖库头文件，以及编译链接选项如宏定义等，有了这些，就可以成功编译

#. 一些公用的script，data文件

安装工具提供的附加值在于package的管理，安装，卸载，版本依赖关系处理，升级更新等。

深入问题:

* easy_install的替代品 `pip <http://pypi.python.org/pypi/pip>`_ ?

* setuptools的bootstrap， 如何自我安装

* distutils的c扩展编译模块

* 是否能将构建，编译，打包与安装分开？只是单纯的下载安装包，解决依赖关系，安装，如apt-get。

* 一般运行 `python setup.py install` ，package就会被安装到运行的那个python。
  如果系统内有多个版本的python，能否运行一个python，为另一个python安装包呢，即
  用 `pythonA setup.py install` 将package安装pythonB的路径？


site.py是什么
---------------------
如果你安装了许多第三方模块，这些包分散在系统的不同地方，那么程序怎么找到这些
模块呢？是，你可以在程序里修改sys.path，但每个程序都这么做，未免有些麻烦。

site.py就是解决这个问题的。它是一个公有库，在python启动时自动加载，分析特定路径
下的.pth文件并自动设置sys.path，你不需要做额外的操作就可以导入第三方模块。

导入site模块::

    jaime@westeros:~/source/Python-2.6.7$ grep -rn site Python
    ...
    Python/pythonrun.c:255:        initsite(); /* Module site */
    Python/pythonrun.c:606:            initsite();
    Python/pythonrun.c:705:/* Import the site module (not into __main__ though) */
    Python/pythonrun.c:708:initsite(void)
    Python/pythonrun.c:711:    m = PyImport_ImportModule("site");
    ...
    jaime@westeros:~/source/Python-2.6.7$ 


    /* Import the site module (not into __main__ though) */

    static void
    initsite(void)
    {
        PyObject *m, *f;
        m = PyImport_ImportModule("site");
        if (m == NULL) {
            f = PySys_GetObject("stderr");
            if (Py_VerboseFlag) {
                PyFile_WriteString(
                    "'import site' failed; traceback:\n", f);
                PyErr_Print();
            }
            else {
                PyFile_WriteString(
                  "'import site' failed; use -v for traceback\n", f);
                PyErr_Clear();
            }
        }
        else {
            Py_DECREF(m);
        }
    }


Lib/site.py::

    PREFIXES = [sys.prefix, sys.exec_prefix]
    
    ...

    def addpackage(sitedir, name, known_paths):
        """Process a .pth file within the site-packages directory:
           For each line in the file, either combine it with sitedir to a path
           and add that to known_paths, or execute it if it starts with 'import '.
        """
        ...
        with f:
            for line in f:
                ...
                line = line.rstrip()
                dir, dircase = makepath(sitedir, line)
                if not dircase in known_paths and os.path.exists(dir):
                    sys.path.append(dir)
                    known_paths.add(dircase)
        if reset:
            known_paths = None
        return known_paths


    def addsitedir(sitedir, known_paths=None):
        """Add 'sitedir' argument to sys.path if missing and handle .pth files in
        'sitedir'"""
        ....
        dotpth = os.extsep + "pth"
        names = [name for name in names if name.endswith(dotpth)]
        for name in sorted(names):
            addpackage(sitedir, name, known_paths)
        if reset:
            known_paths = None
        return known_paths

    def addsitepackages(known_paths):
        """Add site-packages (and possibly site-python) to sys.path"""
        sitedirs = []
        seen = []

        for prefix in PREFIXES:
            if not prefix or prefix in seen:
                continue
            seen.append(prefix)

            if sys.platform in ('os2emx', 'riscos'):
                sitedirs.append(os.path.join(prefix, "Lib", "site-packages"))
            elif os.sep == '/':
                sitedirs.append(os.path.join(prefix, "lib",
                                            "python" + sys.version[:3],
                                            "site-packages"))
                sitedirs.append(os.path.join(prefix, "lib", "site-python"))
            else:
            ...

        for sitedir in sitedirs:
            if os.path.isdir(sitedir):
                addsitedir(sitedir, known_paths)

        return known_paths

    def main():
        global ENABLE_USER_SITE

        abs__file__()
        known_paths = removeduppaths()
        if (os.name == "posix" and sys.path and
            os.path.basename(sys.path[-1]) == "Modules"):
            addbuilddir()
        if ENABLE_USER_SITE is None:
            ENABLE_USER_SITE = check_enableusersite()
        known_paths = addusersitepackages(known_paths)
        known_paths = addsitepackages(known_paths)
        if sys.platform == 'os2emx':
            setBEGINLIBPATH()
        setquit()
        setcopyright()
        sethelper()
        aliasmbcs()
        setencoding()
        execsitecustomize()
        if ENABLE_USER_SITE:
            execusercustomize()
        # Remove sys.setdefaultencoding() so that users cannot change the
        # encoding after initialization.  The test for presence is needed when
        # this module is run as a script, because this code is executed twice.
        if hasattr(sys, "setdefaultencoding"):
            del sys.setdefaultencoding

Bonus，sys.setdefaultencoding在这里被删掉了，系统已经完成初始化，再改变内部编码比较困难。

sys.path 在 removeduppaths 函数中被加入到 known_paths

'site-packages' 目录的具体位置在 addsitepackages 函数中探测， sitedirs取决于PREFIXES，即sys.prefix,
sys.exec_prefix python的安装路径。

.pth 文件的扫描在 addsitedir 中完成，将.pth文件的第三方包目录添加到sys.path则是在 addpackage 。


系统默认2.7python的示例::

    jaime@westeros:~/source/Python/Python-2.6.7$ python
    Python 2.7.1+ (r271:86832, Apr 11 2011, 18:05:24) 
    [GCC 4.5.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import site
    >>> site.__file__
    '/usr/lib/python2.7/site.pyc'
    >>> import sys
    >>> sys.path
    ['', '/usr/local/lib/python2.7/dist-packages/Flask-0.7.2-py2.7.egg',
    '/usr/local/lib/python2.7/dist-packages/Jinja2-2.6-py2.7.egg',
    '/usr/local/lib/python2.7/dist-packages/Werkzeug-0.7.1-py2.7.egg',
    '/usr/local/lib/python2.7/dist-packages/flup-1.0.2-py2.7.egg',
    '/usr/local/lib/python2.7/dist-packages/MySQL_python-1.2.3-py2.7-linux-i686.egg',
    '/usr/lib/python2.7', '/usr/lib/python2.7/plat-linux2',
    '/usr/lib/python2.7/lib-tk', '/usr/lib/python2.7/lib-old',
    '/usr/lib/python2.7/lib-dynload', '/usr/local/lib/python2.7/dist-packages',
    '/usr/lib/python2.7/dist-packages', '/usr/lib/python2.7/dist-packages/PIL',
    '/usr/lib/pymodules/python2.7/gtk-2.0',
    '/usr/lib/python2.7/dist-packages/gst-0.10',
    '/usr/lib/python2.7/dist-packages/gtk-2.0', '/usr/lib/pymodules/python2.7',
    '/usr/lib/pymodules/python2.7/ubuntuone-storage-protocol',
    '/usr/lib/pymodules/python2.7/ubuntuone-control-panel',
    '/usr/lib/pymodules/python2.7/ubuntuone-client']
    >>> sys.prefix
    '/usr'
    >>> sys.executable
    '/usr/bin/python'


    jaime@westeros:~/source/Python/Python-2.6.7$ ls /usr/local/lib/python2.7/dist-packages/
    django                 easy-install.pth       flup-1.0.2-py2.7.egg
    MySQL_python-1.2.3-py2.7-linux-i686.egg
    Django-1.2.7.egg-info  Flask-0.7.2-py2.7.egg  Jinja2-2.6-py2.7.egg
    Werkzeug-0.7.1-py2.7.egg
    jaime@westeros:~/source/Python/Python-2.6.7$ cat /usr/local/lib/python2.7/dist-packages/easy-install.pth 
    import sys; sys.__plen = len(sys.path)
    ./Flask-0.7.2-py2.7.egg
    ./Jinja2-2.6-py2.7.egg
    ./Werkzeug-0.7.1-py2.7.egg
    ./flup-1.0.2-py2.7.egg
    ./MySQL_python-1.2.3-py2.7-linux-i686.egg
    import sys; new=sys.path[sys.__plen:]; del sys.path[sys.__plen:];
    p=getattr(sys,'__egginsert',0); sys.path[p:p]=new; sys.__egginsert =
    p+len(new)
    jaime@westeros:~/source/Python/Python-2.6.7$ 


更多参考:
`Installing Python Modules`_
`Distributing Python Modules`_


.. _Installing Python Modules: http://docs.python.org/release/2.6.7/install/index.html 
.. _Distributing Python Modules: http://docs.python.org/release/2.6.7/distutils/index.html

Python- site-package dirs and .pth files 
http://grahamwideman.wikispaces.com/Python-+site-package+dirs+and+.pth+files


自定义一个package到标准库
------------------------------
直接在Lib/下面加.py文件，make install会自动安装prefix目录。但是如果你要添加目录，
则不会被安装，需要修改Makefile.pre.in::

    jaime@ideer:~/source/Python-2.6.7$ git df
    diff --git a/Makefile.pre.in b/Makefile.pre.in
    index 0329d67..28a17bd 100644
    --- a/Makefile.pre.in
    +++ b/Makefile.pre.in
    @@ -828,7 +828,7 @@ LIBSUBDIRS= lib-tk site-packages test test/output test/data \
                    ctypes ctypes/test ctypes/macholib idlelib idlelib/Icons \
                    distutils distutils/command distutils/tests $(XMLLIBSUBDIRS) \
                    multiprocessing multiprocessing/dummy \
    -               lib-old \
    +               lib-old foo\
                    curses pydoc_data $(MACHDEPS)
     libinstall:    build_all $(srcdir)/Lib/$(PLATDIR)
            @for i in $(SCRIPTDIR) $(LIBDEST); \
    jaime@ideer:~/source/Python-2.6.7$
    jaime@ideer:~/source/Python-2.6.7$ ls Lib/foo/
    bar.py  __init__.py

重新configure, make install。make用LIBSUBDIRS来控制需要复制Lib/下面哪些子目录，
plat-\*平台模块目录在安装时make会自动判断。


从urllib2.urlopen到socket
----------------------------
urlopen::

    _opener = None
    def urlopen(url, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
        global _opener
        if _opener is None:
            _opener = build_opener()
        return _opener.open(url, data, timeout)

urllib2.urlopen共用一个模块变量_opener，也就是install_opener的那个，
搞并发的同学注意了，未知不同请求之间会否相互影响。

urlopen -> build_opener -> OpenerDirector.open, _open, __call_chain__ -> HTTPHandler.http_open ->
AbstractHTTPHandler->do_open -> HTTPConnection.request, _send_request,
send, connect

经过漫长的。。。，鄙人走马观花，自由行的同学可以深入研究:)
终于看到了socket.create_connection, Lib/httplib.py class HTTPConnection::

    def connect(self):
        """Connect to the host and port specified in __init__."""
        self.sock = socket.create_connection((self.host,self.port),
                                             self.timeout)
    ....
    
    def send(self, str):
        """Send `str' to the server."""
        if self.sock is None:
            if self.auto_open:
                self.connect()
            else:
                raise NotConnected()

Lib/socket.py::

        def create_connection(address, timeout=_GLOBAL_DEFAULT_TIMEOUT):
            ....
            msg = "getaddrinfo returns an empty list"
            host, port = address
            for res in getaddrinfo(host, port, 0, SOCK_STREAM):
                af, socktype, proto, canonname, sa = res
                sock = None
                try:
                    sock = socket(af, socktype, proto)
                    if timeout is not _GLOBAL_DEFAULT_TIMEOUT:
                        sock.settimeout(timeout)
                    sock.connect(sa)
                    return sock

在这里，通过getaddrinfo完成dns解析，建了一个socket，sock是内置socketobject类型，
从sock.connect开始，你就潜入C代码的世界了，在 Modules/socketmodule.c +2027::

    static PyObject *
    sock_connect(PySocketSockObject *s, PyObject *addro)
    {
        sock_addr_t addrbuf;
        int addrlen;

费了这半天劲，其实有个简单的方法，你就可以得到这整个的调用路径，yes，万能的raise::

    jaime@ideer:~/source/Python-2.6.7$ git df
    diff --git a/Lib/socket.py b/Lib/socket.py
    index e4f0a81..2a59dd9 100644
    --- a/Lib/socket.py
    +++ b/Lib/socket.py
    @@ -552,6 +552,7 @@ def create_connection(address, timeout=_GLOBAL_DEFAULT_TIMEOUT):
                 if timeout is not _GLOBAL_DEFAULT_TIMEOUT:
                     sock.settimeout(timeout)
                 sock.connect(sa)
    +            raise
                 return sock
     
             except error, msg:

    jaime@ideer:~/source/Python-2.6.7$ ./python
    Python 2.6.7 (r267:88850, Sep  8 2011, 22:55:29) 
    [GCC 4.5.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import urllib2
    >>> urllib2.urlopen('http://douban.com')
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 126, in urlopen
        return _opener.open(url, data, timeout)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 391, in open
        response = self._open(req, data)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 409, in _open
        '_open', req)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 369, in _call_chain
        result = func(*args)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 1181, in http_open
        return self.do_open(httplib.HTTPConnection, req)
      File "/home/chenz/source/Python-2.6.7/Lib/urllib2.py", line 1153, in do_open
        h.request(req.get_method(), req.get_selector(), req.data, headers)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 914, in request
        self._send_request(method, url, body, headers)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 951, in _send_request
        self.endheaders()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 908, in endheaders
        self._send_output()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 780, in _send_output
        self.send(msg)
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 739, in send
        self.connect()
      File "/home/chenz/source/Python-2.6.7/Lib/httplib.py", line 720, in connect
        self.timeout)
      File "/home/chenz/source/Python-2.6.7/Lib/socket.py", line 555, in create_connection
        raise
    TypeError: exceptions must be old-style classes or derived from BaseException, not NoneType
    >>> 


urllib2.py OpenerDirector的open函数::

        def open(self, fullurl, data=None, timeout=socket._GLOBAL_DEFAULT_TIMEOUT):
                # accept a URL or a Request object
                if isinstance(fullurl, basestring):
                    req = Request(fullurl, data)
                else:
                    req = fullurl
                    if data is not None:
                        req.add_data(data)

                req.timeout = timeout
                protocol = req.get_type()

                # pre-process request
                meth_name = protocol+"_request"
                for processor in self.process_request.get(protocol, []):
                    meth = getattr(processor, meth_name)
                    req = meth(req)

                response = self._open(req, data)

                # post-process response
                meth_name = protocol+"_response"
                for processor in self.process_response.get(protocol, []):
                    meth = getattr(processor, meth_name)
                    response = meth(req, response)

                return response

涵盖了一个http请求的全部过程，创建Request对象，获得协议类型，对请求进行预处理如
header，认证等，打开连接，处理响应，错误处理等，值得细究。


urllib2中的重定向
---------------------
http_response负责对服务器响应进行处理。如果状态码如果不是2xx，则启动错误处理机制::

    class HTTPErrorProcessor(BaseHandler):
        """Process HTTP error responses."""
        handler_order = 1000  # after all other processing

        def http_response(self, request, response):
            code, msg, hdrs = response.code, response.msg, response.info()

            # According to RFC 2616, "2xx" code indicates that the client's
            # request was successfully received, understood, and accepted.
            if not (200 <= code < 300):
                response = self.parent.error(
                    'http', request, response, code, msg, hdrs)

            return response

        https_response = http_response


3xx重定向指令由HTTPRedirectHandler负责，具体函数为http_error_3xx，主要做一些外围性
检查，分析获取重定向的地址，检测协议和循环重定向。如果一切ok，则调用redirect_request
生成新的Request对象，传给parent opener执行这个新req。一切又回到了开始。


start_response和exc_info
------------------------------

`WSGI`_ 规定了两个函数, write 和start_response::

    def start_response(status, response_headers, exc_info=None):

start_response返回write函数。这是为了和惯于用print类的应用进些兼容。
wsgi的application默认返回iterable，含有所有要输出的内容，server遍历它，
完成真正的输出::


 result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()

write函数一旦被调用，就会自动激活header的输出，所以调用write是你改变header的
最后机会。

exc_info主要用于对异常进些处理，pep333中的示例代码::

    try:
        # regular application code here
        status = "200 Froody"
        response_headers = [("content-type", "text/plain")]
        start_response(status, response_headers)
        return ["normal body goes here"]
    except:
        # XXX should trap runtime issues like MemoryError, KeyboardInterrupt
        #     in a separate handler before this bare 'except:'...
        status = "500 Oops"
        response_headers = [("content-type", "text/plain")]
        start_response(status, response_headers, sys.exc_info())
        return ["error body goes here"]

异常发生时，如果：

* 200 OK没有被发送，没有调用过write，或者应用返回的iteralbe内容server还没有开始
  发送，总之，header没有发出，此时还有挽救的余地，将状态码改为500，忽略掉exc_info，
  用户自定义的错误信息，debug堆栈信息可以在error body里面输出。

* 200 OK这个header已经被server发送给客户端，已经发送了部分后续body内容，此时程序抛出
  异常，application探测到错误，怎么办？再发送500 Oops状态码也无济于事，wsgi server
  能做的只是raise exc_info，把事情搞大，捅到上层去。wsgi规定用户不可以捕捉带有exc_info
  信息的start_response抛出的异常。

start_response对这两种情况提供了一种统一的处理方式。在cgi环境里运行的wsgi start_response::

  def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # Re-raise original exception if headers sent
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write


复杂的代码，不知道异常抛出时的准确状态，此为start_response exc_info的目的，可以用try except
把application的整个逻辑保护起来。或者你本就不该写复杂的代码？笑:) 或许你可以精巧的构造异常
处理代码，将header是否发送区分开来？

http协议的状态码status 200表示资源找到，但是后续处理出问题，怎么办？是否可以加一些位于最后的header，
表示请求成功完成？这样即使header已经发送，也可以做些别的措施暗示请求出错。content-length
是否起到了这样的作用？这也许是属于不同层的问题。

是否可以改变应用逻辑，全部处理完毕后一起发送header和body？区分应用相关，数据量大或长时间的应用
如何处理？stream？

.. _`WSGI`: http://www.python.org/dev/peps/pep-0333/

builtin的函数在哪
-----------------------
__builtin__ 模块对应的c文件是Python/bltinmodule.c::

    static PyMethodDef builtin_methods[] = {
        {"__import__",      (PyCFunction)builtin___import__, METH_VARARGS | METH_KEYWORDS, import_doc},
        {"abs",             builtin_abs,        METH_O, abs_doc},
        ...
        {"dir",             builtin_dir,        METH_VARARGS, dir_doc},
        {"divmod",          builtin_divmod,     METH_VARARGS, divmod_doc},
     
dir, I saw you! 这就是python dir函数的入口，对应的c代码为builtin_dir::

        static PyObject *
        builtin_dir(PyObject *self, PyObject *args)
        {
            PyObject *arg = NULL;

            if (!PyArg_UnpackTuple(args, "dir", 0, 1, &arg))
                return NULL;
            return PyObject_Dir(arg);
        }

进行简单的参数处理，获得参数object的指针，然后调用该object自身的dir处理函数，simple。
至于PyObject_Dir如何工作，则为后话了。现在不妨翻看一下其他的builtin函数代码。

PyArg_UnpackTuple 参数分析

+ args 是从python上层传过来的参数tuple
  
+ "dir" 用于出错时显示哪个函数::

    >>> dir(1, 2)
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    TypeError: dir expected at most 1 arguments, got 2

+ 0表示参数个数最少为0，1表示最多为1
  
+ &arg 提取到的参数存放在这里


METH_O 表示该函数只有一个参数，METH_VARARGS表示参数个数可变，具体定义在Include/methodobject.h::

    jaime@ideer:~/source/Python-2.6.7$ grep -rn METH_O Include/
    Include/methodobject.h:53:#define METH_OLDARGS  0x0000
    Include/methodobject.h:56:/* METH_NOARGS and METH_O must not be combined with the flags above. */
    Include/methodobject.h:58:#define METH_O        0x0008
    jaime@ideer:~/source/Python-2.6.7$ grep -rn METH_O Python/
    ...
    Python/ceval.c:3730:        if (flags & (METH_NOARGS | METH_O)) {
    Python/ceval.c:3736:            else if (flags & METH_O && na == 1) {
    jaime@ideer:~/source/Python-2.6.7$ 

在builtin_methods数组中只是声明了一下，运行时的参数检查在Python/ceval.c +3729 完成::


    PCALL(PCALL_CFUNCTION);
    if (flags & (METH_NOARGS | METH_O)) {
        PyCFunction meth = PyCFunction_GET_FUNCTION(func);
        PyObject *self = PyCFunction_GET_SELF(func);
        if (flags & METH_NOARGS && na == 0) {
            C_TRACE(x, (*meth)(self,NULL));
        }
        else if (flags & METH_O && na == 1) {
            PyObject *arg = EXT_POP(*pp_stack);
            C_TRACE(x, (*meth)(self,arg));
            Py_DECREF(arg);
        }
        else {
            err_args(func, flags, na);
            x = NULL;
        }
    }

如果定义了METH_NOARGS或METH_O，但是参数个数na又不为0或1，则通过err_args报错。

Python/ceval.c +3661::

    static void
    err_args(PyObject *func, int flags, int nargs)
    {
        if (flags & METH_NOARGS)
            PyErr_Format(PyExc_TypeError,
                         "%.200s() takes no arguments (%d given)",
                         ((PyCFunctionObject *)func)->m_ml->ml_name,
                         nargs);
        else
            PyErr_Format(PyExc_TypeError,
                         "%.200s() takes exactly one argument (%d given)",
                         ((PyCFunctionObject *)func)->m_ml->ml_name,
                         nargs);
    }


Hello, exception! 第一个异常
------------------------------

Modules/posixmodule.c +6313::

    static PyObject *
    posix_open(PyObject *self, PyObject *args)
    {
        char *file = NULL;
        int flag;
        int mode = 0777;
        int fd;

    #ifdef MS_WINDOWS
        if (unicode_file_names()) {
            PyUnicodeObject *po;
            if (PyArg_ParseTuple(args, "Ui|i:mkdir", &po, &flag, &mode)) {
                Py_BEGIN_ALLOW_THREADS
                /* PyUnicode_AS_UNICODE OK without thread
                   lock as it is a simple dereference. */
                fd = _wopen(PyUnicode_AS_UNICODE(po), flag, mode);
                Py_END_ALLOW_THREADS
                if (fd < 0)
                    return posix_error();
                return PyInt_FromLong((long)fd);
            }
            /* Drop the argument parsing error as narrow strings
               are also valid. */
            PyErr_Clear();
        }
    #endif

        if (!PyArg_ParseTuple(args, "eti|i",
                              Py_FileSystemDefaultEncoding, &file,
                              &flag, &mode))
            return NULL;

        Py_BEGIN_ALLOW_THREADS
        fd = open(file, flag, mode);
        Py_END_ALLOW_THREADS
        if (fd < 0)
            return posix_error_with_allocated_filename(file);
        PyMem_Free(file);
        return PyInt_FromLong((long)fd);
    }

前半部分代码是windows用的，linux的在后半部。先获得参数: file, flag,
可选的mode。然后调用open系统函数，最后返回一个Int类型的python对象。

仔细观察，如果参数有错误，返回NULL，在python层面则表现为抛出了异常，
由此是否可以猜测，对于此函数来说，返回值为NULL就表示有异常？还有什么要注意的吗？

再看，如果是文件不存在，open失败，同样在上层表现为异常，但是返回前的处理却不一样::

    static PyObject *
    posix_error_with_allocated_filename(char* name)
    {
        PyObject *rc = PyErr_SetFromErrnoWithFilename(PyExc_OSError, name);
        PyMem_Free(name);
        return rc;
    }

可以看出，open之前，file还是一个空指针，没有指向分配的内存，所以只返回NULL就足够了。
open之后，不管是成功还是失败，file指针都需要被释放掉。这是需要特别小心的地方，一旦
处理不到，就会造成内存泄露。原则是，在返回之前，一定要把已申请的资源处理好。

现在有了足够的信心，照着原有代码的例子，我们可以抛出自己的异常。用什么函数呢？
PyErr_SetFromErrnoWithFilename 看着像和异常有关，翻看代码，可以看到类似函数::

    +2282
    if (len >= MAX_PATH) {
        PyErr_SetString(PyExc_ValueError, "path too long");
        return NULL;
    }

    +2831
    else if (!PyTuple_Check(arg) || PyTuple_Size(arg) != 2) {
        PyErr_SetString(PyExc_TypeError,
                        "utime() arg 2 must be a tuple (atime, mtime)");
        goto done;
    }
 
PyErr_SetString 抛出一个纯c字符串，不需要担心对象引用，正是我们想要的。第一个
参数为异常的类型。

file是 `char *` 类型，这意味是我们可以用strcmp。

代码如下::

    jaime@ideer:~/source/Python-2.6.7$ git df
    diff --git a/Modules/posixmodule.c b/Modules/posixmodule.c
    index 822bc11..7501f0d 100644
    --- a/Modules/posixmodule.c
    +++ b/Modules/posixmodule.c
    @@ -6337,11 +6337,19 @@ posix_open(PyObject *self, PyObject *args)
         }
     #endif
     
    +    printf("Entering posix_open\n");
    +
         if (!PyArg_ParseTuple(args, "eti|i",
                               Py_FileSystemDefaultEncoding, &file,
                               &flag, &mode))
             return NULL;
     
    +    if (strcmp(file, "hello") == 0) {
    +        PyErr_SetString(PyExc_ValueError, "Hello, exception!");
    +        PyMem_Free(file);
    +        return NULL;
    +    }
    +
         Py_BEGIN_ALLOW_THREADS
         fd = open(file, flag, mode);
         Py_END_ALLOW_THREADS
    jaime@ideer:~/source/Python-2.6.7$


输出::

    jaime@ideer:~/source/Python-2.6.7$ ./python 
    Python 2.6.7 (r267:88850, Sep 10 2011, 12:12:00) 
    [GCC 4.5.2] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import os
    >>> os.open()
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    TypeError: function takes at least 2 arguments (0 given)
    >>> os.open('hello', os.O_RDONLY)
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    ValueError: Hello, exception!
    >>> os.open('test', os.O_RDONLY)
    Entering posix_open
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    OSError: [Errno 2] No such file or directory: 'test'
    >>> os.open('test', os.O_WRONLY | os.O_CREAT)
    Entering posix_open
    3
    >>> 

注意三个异常发生的时刻，以及类型TypeError, ValueError,
OSError。另一个有趣的函数是 PyErr_Format，可以抛出一个格式化的字符串。

Python/builtinmodule.c +188::

    if (kwdict != NULL && !PyDict_Check(kwdict)) {
        PyErr_Format(PyExc_TypeError,
                     "apply() arg 3 expected dictionary, found %s",
                     kwdict->ob_type->tp_name);
        goto finally;
    }
 
更多异常处理函数参见 Include/pyerrors.h, Python/errors.c。

PyArg_ParseTuple 参见 The Python/C API。

sys.path, sys.prefix, sys.exec_prefix, PYTHONHOME 和 PYTHONPATH
------------------------------------------------------------------
sys模块代码在 Python/sysmodule.c::

    PyObject *
    _PySys_Init(void)
    {
        PyObject *m, *v, *sysdict;
        PyObject *sysin, *sysout, *syserr;
        char *s;

        m = Py_InitModule3("sys", sys_methods, sys_doc);
        if (m == NULL)
            return NULL;
        sysdict = PyModule_GetDict(m);
    #define SET_SYS_FROM_STRING(key, value)                 \
        v = value;                                          \
        if (v != NULL)                                      \
            PyDict_SetItemString(sysdict, key, v);          \
        Py_XDECREF(v)
        ...

        SET_SYS_FROM_STRING("executable",
                    PyString_FromString(Py_GetProgramFullPath()));
        SET_SYS_FROM_STRING("prefix",
                    PyString_FromString(Py_GetPrefix()));
        SET_SYS_FROM_STRING("exec_prefix",
                    PyString_FromString(Py_GetExecPrefix()));
        ...

    #undef SET_SYS_FROM_STRING
        if (PyErr_Occurred())
            return NULL;
        return m;
        }

在 Py_InitializeEx 中，builtin模块之后初始化, Python/pythonrun.c::

   bimod = _PyBuiltin_Init();
    if (bimod == NULL)
        Py_FatalError("Py_Initialize: can't initialize __builtin__");
    interp->builtins = PyModule_GetDict(bimod);
    if (interp->builtins == NULL)
        Py_FatalError("Py_Initialize: can't initialize builtins dict");
    Py_INCREF(interp->builtins);

    sysmod = _PySys_Init();
    ...
    PySys_SetPath(Py_GetPath());
    PyDict_SetItemString(interp->sysdict, "modules",
                         interp->modules);

PySys_SetPath 负责生成sys.path 这个python object.

sys.prefix, sys.exec_prefix, sys.executable, 还有sys.path 都和getpath.c有关。

PYTHONHOME和progname, Python/pythonrun.c::

    static char *progname = "python";

    void
    Py_SetProgramName(char *pn)
    {
        if (pn && *pn)
            progname = pn;
    }

    char *
    Py_GetProgramName(void)
    {
        return progname;
    }

    static char *default_home = NULL;

    void
    Py_SetPythonHome(char *home)
    {
        default_home = home;
    }

    char *
    Py_GetPythonHome(void)
    {
        char *home = default_home;
        if (home == NULL && !Py_IgnoreEnvironmentFlag)
            home = Py_GETENV("PYTHONHOME");
        return home;
    }

如果是命令行启动，progname被设置为argv[0], Modules/main.c::
    
    Py_SetProgramName(argv[0]);

常量定义 Modules/getpath.c::

    /* 路径定义*/
    static char prefix[MAXPATHLEN+1];     /* sys.prefix */
    static char exec_prefix[MAXPATHLEN+1];  /* sys.exec_prefix */
    static char progpath[MAXPATHLEN+1];     /* sys.executable */
    static char *module_search_path = NULL;  /* sys.path */
    static char lib_python[] = "lib/python" VERSION;

    #ifndef PREFIX
    #  ifdef __VMS
    #    define PREFIX ""
    #  else
    /* configure不指定时，默认为/usr/local。这个make install的默认不一样?*/
    #    define PREFIX "/usr/local"
    #  endif
    #endif

    #ifndef EXEC_PREFIX
    #define EXEC_PREFIX PREFIX
    #endif

    #ifndef PYTHONPATH
    #define PYTHONPATH PREFIX "/lib/python" VERSION ":" \
                  EXEC_PREFIX "/lib/python" VERSION "/lib-dynload"
    #endif

    #ifndef LANDMARK
    #define LANDMARK "os.py"
    #endif
    ...

Makefile中定义有PYTHONPATH::

    PYTHONPATH=$(COREPYTHONPATH)
    COREPYTHONPATH=$(DESTPATH)$(SITEPATH)$(TESTPATH)$(MACHDEPPATH)$(EXTRAMACHDEPPATH)$(TKPATH)$(OLDPATH)
    OLDPATH=:lib-old
    TKPATH=:lib-tk
    EXTRAMACHDEPPATH=
    MACHDEPPATH=:plat-$(MACHDEP)
    TESTPATH=
    SITEPATH=
    DESTPATH=
    MACHDESTLIB=$(BINLIBDEST)
    DESTLIB=$(LIBDEST)
    ...

    Modules/getpath.o: $(srcdir)/Modules/getpath.c Makefile
        $(CC) -c $(PY_CFLAGS) -DPYTHONPATH='"$(PYTHONPATH)"' \
            -DPREFIX='"$(prefix)"' \
            -DEXEC_PREFIX='"$(exec_prefix)"' \
            -DVERSION='"$(VERSION)"' \
            -DVPATH='"$(VPATH)"' \
            -o $@ $(srcdir)/Modules/getpath.c

最主要的路径分析函数calculate_path，该函数的注释非常详尽，可仔细阅读，其只要目的是根据
argv, $PYTHONHOME, $PYTHONPATH推测progpath, prefix, exec_prefix, module_search_path::

    static void
    calculate_path(void)
    {
        extern char *Py_GetProgramName(void);

        static char delimiter[2] = {DELIM, '\0'};
        static char separator[2] = {SEP, '\0'};
        char *pythonpath = PYTHONPATH;
        char *rtpypath = Py_GETENV("PYTHONPATH");
        char *home = Py_GetPythonHome();
        char *path = getenv("PATH");
        char *prog = Py_GetProgramName();
        char argv0_path[MAXPATHLEN+1];
        char zip_path[MAXPATHLEN+1];
        int pfound, efound; /* 1 if found; -1 if found build directory */
        char *buf;
        size_t bufsz;
        size_t prefixsz;
        char *defpath = pythonpath;
        ...

            /* 1.1 寻找progpath */
            if (strchr(prog, SEP))
                    strncpy(progpath, prog, MAXPATHLEN); /* 如果argv[0]含有分割符'/' */
            ...
            else if (path) {
                    /* 没有'/'，可能是嵌入或直接运行 python 命令
                     在 PATH中 查找 prog*/

                    /* 如果是嵌入的python，则使用PATH搜到的python可执行文件可能并不是你所要的
                    此时最好明确指定 PYTHONHOME。*/
                    ...
            }
            else
                    progpath[0] = '\0'; /* PATH 为空*/

            if (progpath[0] != SEP && progpath[0] != '\0')
                    absolutize(progpath);
            /* 1.2 设置argv0_path */
            strncpy(argv0_path, progpath, MAXPATHLEN);
            argv0_path[MAXPATHLEN] = '\0';
    ...

    #if HAVE_READLINK
        /* 转换符号连接 */
        ...
    #endif /* HAVE_READLINK */

        reduce(argv0_path); /* dirname，转换为目录 */

        /* 2. 根据标志性文件，查找prefix路径*/
        if (!(pfound = search_for_prefix(argv0_path, home))) {
            if (!Py_FrozenFlag)
                fprintf(stderr,
                    "Could not find platform independent libraries <prefix>\n");
            /* ismodule 返回false的情况，直接使用PREFIX */
            strncpy(prefix, PREFIX, MAXPATHLEN);
            joinpath(prefix, lib_python);
        }
        else
            reduce(prefix); 

        /* 3. 获取python26.zip路径，在后面添加到sys.path */
        strncpy(zip_path, prefix, MAXPATHLEN);
        ...
        bufsz = strlen(zip_path);   /* Replace "00" with version */
        zip_path[bufsz - 6] = VERSION[0];
        zip_path[bufsz - 5] = VERSION[2];

        /* 4. 查找 exec_prefix */
        if (!(efound = search_for_exec_prefix(argv0_path, home))) {
            if (!Py_FrozenFlag)
                fprintf(stderr,
                    "Could not find platform dependent libraries <exec_prefix>\n");
            strncpy(exec_prefix, EXEC_PREFIX, MAXPATHLEN);
            joinpath(exec_prefix, "lib/lib-dynload");
        }
        /* If we found EXEC_PREFIX do *not* reduce it!  (Yet.) */

        if ((!pfound || !efound) && !Py_FrozenFlag)
            fprintf(stderr,
                    "Consider setting $PYTHONHOME to <prefix>[:<exec_prefix>]\n");

        /* sys.path中的路径临时存放在 rtpython 中, 注意宏PYTHONPATH 和环境变量
        PYTHONPATH*/

        /* Calculate size of return buffer.
         */
        bufsz = 0;

        if (rtpypath)
            bufsz += strlen(rtpypath) + 1;

        prefixsz = strlen(prefix) + 1;

        while (1) {
            char *delim = strchr(defpath, DELIM);
            /* 计算宏定义PYTHONPATH中的路径size*/

        }

        bufsz += strlen(zip_path) + 1;
        bufsz += strlen(exec_prefix) + 1;


        /* 5. 计算module_search_path即sys.path*/

        /* This is the only malloc call in this file */
        buf = (char *)PyMem_Malloc(bufsz);

        if (buf == NULL) {
            /* We can't exit, so print a warning and limp along */
            fprintf(stderr, "Not enough memory for dynamic PYTHONPATH.\n");
            fprintf(stderr, "Using default static PYTHONPATH.\n");
            module_search_path = PYTHONPATH;
        }
        else {
            /* Run-time value of $PYTHONPATH goes first */
            /* 5.1 PYTHONPATH 环境变量*/
            if (rtpypath) {
                strcpy(buf, rtpypath);
                strcat(buf, delimiter);
            }
            else
                buf[0] = '\0';

            /* Next is the default zip path */
            /* 5.2 python26.zip*/
            strcat(buf, zip_path);
            strcat(buf, delimiter);

            /* Next goes merge of compile-time $PYTHONPATH with
             * dynamically located prefix.
             */
            defpath = pythonpath;
            while (1) {
                char *delim = strchr(defpath, DELIM);
                /* 5.3 宏定义 PYTHONPATH */

            }
            strcat(buf, delimiter);
            
            /* 5.4 Finally, on goes the directory for dynamic-load modules */
            strcat(buf, exec_prefix);

            /* And publish the results */
            module_search_path = buf;
        }

        /* 6. 善后处理 prefix, exec_prefix
           分别从/usr/local/lib/python2.6, /usr/local/lib/python2.6/lib-dynload
           转换为/usr/local, /usr/local
        */
        /* Reduce prefix and exec_prefix to their essence,
         * e.g. /usr/local/lib/python1.5 is reduced to /usr/local.
         * If we're loading relative to the build directory,
         * return the compiled-in defaults instead.
         */
        if (pfound > 0) {
            reduce(prefix);
            reduce(prefix);
            /* The prefix is the root directory, but reduce() chopped
             * off the "/". */
            if (!prefix[0])
                    strcpy(prefix, separator);
        }
        else
            strncpy(prefix, PREFIX, MAXPATHLEN);

        if (efound > 0) {
            reduce(exec_prefix);
            reduce(exec_prefix);
            reduce(exec_prefix);
            if (!exec_prefix[0])
                    strcpy(exec_prefix, separator);
        }
        else
            strncpy(exec_prefix, EXEC_PREFIX, MAXPATHLEN);
    }

search_for_prefix::

    /* 当此函数成功返回时，prefix存放有LANDMARK文件的完整路径*/
    static int
    search_for_prefix(char *argv0_path, char *home)
    {
        size_t n;
        char *vpath;

        /* If PYTHONHOME is set, we believe it unconditionally */
        /* PYTHONHOME优先级最高 */
        if (home) {
            char *delim;
            strncpy(prefix, home, MAXPATHLEN);
            delim = strchr(prefix, DELIM);
            if (delim)
                *delim = '\0';
            joinpath(prefix, lib_python);
            joinpath(prefix, LANDMARK); /* prefix现在是os.py的完整路径 */
            return 1;
        }

        /* Check to see if argv[0] is in the build directory */
        strcpy(prefix, argv0_path);
        joinpath(prefix, "Modules/Setup");
        if (isfile(prefix)) {
            /* Check VPATH to see if argv0_path is in the build directory. */
            vpath = VPATH;
            strcpy(prefix, argv0_path);
            joinpath(prefix, vpath);
            joinpath(prefix, "Lib");
            joinpath(prefix, LANDMARK);
            if (ismodule(prefix))
                return -1; /* 返回-1 特殊情况表示源码目录*/
        }

        /* Search from argv0_path, until root is found */
        /* 递归向上寻找LANDMARK文件 */
        copy_absolute(prefix, argv0_path);
        do {
            n = strlen(prefix);
            joinpath(prefix, lib_python);
            joinpath(prefix, LANDMARK);
            if (ismodule(prefix))
                return 1;
            prefix[n] = '\0';
            reduce(prefix);
        } while (prefix[0]);

        /* 没找到，试试预定义的PREFIX? */
        /* Look at configure's PREFIX */
        strncpy(prefix, PREFIX, MAXPATHLEN);
        joinpath(prefix, lib_python);
        joinpath(prefix, LANDMARK);
        if (ismodule(prefix))
            return 1;

        /* Fail */
        return 0;
    }

search_for_exec_prefix和search_for_prefix类似, 只不过标志性文件为lib-dynload，
如果没开启动态模块加载， 这个目录可能不存在，需要注意。

在calculate_path做了繁重的准备之后, Py_GetProgramFullPath，Py_GetPrefix,
Py_GetExecPrefix 只是简单的封装，在此就不赘述了。

轻松时刻哈哈，判断python模块还可以更偷懒吗::

    static int
    ismodule(char *filename)        /* Is module -- check for .pyc/.pyo too */
    {
        if (isfile(filename))
            return 1;

        /* Check for the compiled version of prefix. */
        if (strlen(filename) < MAXPATHLEN) {
            strcat(filename, Py_OptimizeFlag ? "o" : "c");
            if (isfile(filename))
                return 1;
        }
        return 0;
    }


怎么指定要内建builtin的模块
-----------------------------------
Modules/Setup.dist 控制将要内建到python中的模块列表

如果Setup文件不存在，make命令会将Setup.dist复制为Setup文件。但是一旦存在, 则
不会在复制，故修改Setup.dist后，必须手动复制为Setup方能生效，或者你可以直接
修改Setup文件。

todo: 进一步分析如何完成链接


os模块
------
Linux平台os模块函数多从posix模块中导入，代码在 Modules/posixmodule.c::

    _names = sys.builtin_module_names

    if 'posix' in _names:
        name = 'posix'
        linesep = '\n'
        from posix import *
        try:
            from posix import _exit
        except ImportError:
            pass
        import posixpath as path

        import posix
        __all__.extend(_get_exports_list(posix))
        del posix

所以os.open实际上是posix.open::

    >>> import os
    >>> import posix
    >>> id(os.open)
    3077348460L
    >>> id(posix.open)
    3077348460L
    >>>

os.path实际上posixpath模块的一个别名，代码在Lib/posixpath.py。

其他平台有nt，os2等模块，这些才是真正的底层实现。os模块只提供跨平台封装。


多版本python的一些信息
--------------------------

python运行需要的信息如下：

* 可执行文件python

* .py标准库，.so c扩展所在位置

* 第三方package，你在程序中导入的非标准库位置

* 用户编写的.py文件位置

知道以上信息，就可构建一个完整的python运行环境。


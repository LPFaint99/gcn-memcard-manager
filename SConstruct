# -*- python -*-

import os
import sys
import platform

# Home made tests
sys.path.append('SconsTests')
import wxconfig
import utils

# Some features need at least SCons 1.2
EnsureSConsVersion(1, 2)

warnings = [
    'all',
    'write-strings',
    'shadow',
    'pointer-arith',
    'packed',
    'no-conversion',
    ]
compileFlags = [
    '-fno-exceptions',
    '-fno-strict-aliasing',
    '-msse2',
    ]
if sys.platform != 'win32':
    compileFlags += [ '-fvisibility=hidden' ]

cppDefines = [
    ( '_FILE_OFFSET_BITS', 64),
    '_LARGEFILE_SOURCE',
    'GCC_HASCLASSVISIBILITY',
    ]

basedir = os.getcwd()+ '/'

include_paths = [
    basedir + 'Externals/Common_Dolphin@r6316/Src',
    basedir + 'GCN_Memcard_Manager/Src',
  ]

builders = {}
if sys.platform == 'darwin':
    from plistlib import writePlist
    def createPlist(target, source, env):
        properties = {}
        for srcNode in source:
            properties.update(srcNode.value)
            for dstNode in target:
                writePlist(properties, str(dstNode))
    builders['Plist'] = Builder(action = createPlist)

# Handle command line options
vars = Variables('args.cache')

vars.AddVariables(
    BoolVariable('verbose', 'Set for compilation line', False),
    BoolVariable('lint', 'Set for lint build (extra warnings)', False),
    EnumVariable('flavor', 'Choose a build flavor', 'release',
                 allowed_values = ('release','devel','debug','fastlog','prof'),
                 ignorecase = 2
                 ),
    PathVariable('wxconfig', 'Path to the wxconfig', None),
    ('CC', 'The c compiler', 'gcc'),
    ('CXX', 'The c++ compiler', 'g++'),
    )

if sys.platform == 'win32':
    env = Environment(
        CPPPATH = include_paths,
        RPATH = [],
        LIBS = [],
        LIBPATH = [],
        tools = [ 'mingw' ],
        variables = vars,
        ENV = os.environ,
        BUILDERS = builders,
        )
else:
    env = Environment(
        CPPPATH = include_paths,
        RPATH = [],
        LIBS = [],
        LIBPATH = [],
        variables = vars,
        ENV = {
            'PATH' : os.environ['PATH'],
            'HOME' : os.environ['HOME']
        },
        BUILDERS = builders,
        )

# Save the given command line options
vars.Save('args.cache', env)

# Verbose compile
if not env['verbose']:
    env['CCCOMSTR'] = "Compiling $TARGET"
    env['CXXCOMSTR'] = "Compiling $TARGET"
    env['ARCOMSTR'] = "Archiving $TARGET"
    env['LINKCOMSTR'] = "Linking $TARGET"
    env['ASCOMSTR'] = "Assembling $TARGET"
    env['ASPPCOMSTR'] = "Assembling $TARGET"
    env['SHCCCOMSTR'] = "Compiling shared $TARGET"
    env['SHCXXCOMSTR'] = "Compiling shared $TARGET"
    env['SHLINKCOMSTR'] = "Linking shared $TARGET"
    env['RANLIBCOMSTR'] = "Indexing $TARGET"

# Build flavour
flavour = env['flavor']
if (flavour == 'debug'):
    compileFlags.append('-ggdb')
    cppDefines.append('_DEBUG') #enables LOGGING
    # FIXME: this disable wx debugging how do we make it work?
    cppDefines.append('NDEBUG')
elif (flavour == 'devel'):
    compileFlags.append('-ggdb')
elif (flavour == 'fastlog'):
    compileFlags.append('-O3')
    cppDefines.append('DEBUGFAST')
elif (flavour == 'prof'):
    compileFlags.append('-O3')
    compileFlags.append('-ggdb')
elif (flavour == 'release'):
    compileFlags.append('-O3')

# More warnings
if env['lint']:
    warnings.append('error')
    # Should check for the availability of these (in GCC 4.3 or newer)
    if sys.platform != 'darwin':
        warnings.append('no-array-bounds')
        warnings.append('no-unused-result')
    # wxWidgets causes too many warnings with these
    #warnings.append('unreachable-code')
    #warnings.append('float-equal')

# Add the warnings to the compile flags
compileFlags += [ '-W' + warning for warning in warnings ]

env['CCFLAGS'] = compileFlags
if sys.platform == 'win32':
    env['CXXFLAGS'] = compileFlags
else:
    env['CXXFLAGS'] = compileFlags + [ '-fvisibility-inlines-hidden' ]
env['CPPDEFINES'] = cppDefines

# Configuration tests section
tests = {'CheckWXConfig' : wxconfig.CheckWXConfig,
         'CheckPKGConfig' : utils.CheckPKGConfig,
         'CheckPKG' : utils.CheckPKG,
         }

# Object files
#VariantDir(env['build_dir'], '.', duplicate=0)
env['build_dir'] = os.path.join(basedir, 'Build',
    platform.system() + '-' + 'MemCardManager' + os.sep)

conf = env.Configure(config_h="Externals/Common_Dolphin@r6316/Src/Config.h", custom_tests = tests)

if not conf.CheckPKGConfig('0.15.0'):
    print "Can't find pkg-config, some tests will fail"

# OS X specifics
if sys.platform == 'darwin':
    env['HAVE_X11'] = 0
    compileFlags.append('-mmacosx-version-min=10.5')
    conf.Define('MAP_32BIT', 0)
else:
    env['HAVE_X11'] = conf.CheckPKG('x11')

# Handling wx flags CCFLAGS should be created before
wxmods = ['adv', 'core', 'base']
env['USE_WX'] = 0

env['HAVE_WX'] = conf.CheckWXConfig('2.8', wxmods, 0)

conf.Define('HAVE_WX', env['HAVE_WX'])
conf.Define('USE_WX', env['USE_WX'])
conf.Define('HAVE_X11', env['HAVE_X11'])

# Profile
env['USE_OPROFILE'] = 0
if (flavour == 'prof'):
    proflibs = [ '/usr/lib/oprofile', '/usr/local/lib/oprofile' ]
    env['LIBPATH'].append(proflibs)
    env['RPATH'].append(proflibs)
    if conf.CheckPKG('opagent'):
        env['USE_OPROFILE'] = 1
    else:
        print "Can't build prof without oprofile, disabling"

conf.Define('USE_OPROFILE', env['USE_OPROFILE'])
# After all configuration tests are done
conf.Finish()

if env['HAVE_WX']:
    wxconfig.ParseWXConfig(env)
else:
    print "WX not found or disabled, not building GUI"

# Install paths
extra=''
if flavour == 'debug':
    extra = '-debug'
elif flavour == 'prof':
    extra = '-prof'

# TODO: support global install
env['prefix'] = os.path.join(basedir + 'Binary',
    'MemCardManager' + extra + os.sep)
# TODO: add bin
env['binary_dir'] = env['prefix']

# Static libs goes here
env['local_libs'] =  env['build_dir'] + os.sep + 'libs' + os.sep

env['LIBPATH'].append(env['local_libs'])

# Die on unknown variables
unknown = vars.UnknownVariables()
if unknown:
    print "Unknown variables:", unknown.keys()
    Exit(1)


Export('env')

dirs = [
    basedir + 'Externals/Common_Dolphin@r6316/Src',
    basedir + 'GCN_Memcard_Manager/Src',
    ]

#for subdir in dirs:
#    SConscript(
#        subdir + os.sep + 'SConscript',
#        variant_dir = env[ 'build_dir' ] + subdir + os.sep,
#        duplicate=0
#        )
# Now that platform configuration is done, propagate it to modules
for subdir in dirs:
    SConscript(dirs = subdir, duplicate = 0, exports = 'env',
        variant_dir = env['build_dir'] + os.sep + subdir)
    if subdir.startswith('GCN_Memcard_Manager'):
        env['CPPPATH'] += ['#' + subdir]

# Print a nice progress indication when not compiling
Progress(['-\r', '\\\r', '|\r', '/\r'], interval = 5)

# Generate help
Help(vars.GenerateHelpText(env))

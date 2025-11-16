#!/usr/bin/env python
import os
import sys
import shutil
import glob

from methods import print_error


libname = "Test_extension"
projectdir = "demo"
addon_name = "example_plugin"

# Define external libraries to link and copy (without extension)
# The script will automatically find the right platform-specific file
external_libs = []

localEnv = Environment(tools=["default"], PLATFORM="")

def copy_folder(from_path, to_path):
    if not os.path.exists(to_path):
        os.makedirs(to_path)
    shutil.rmtree(to_path, ignore_errors=True)
    shutil.copytree(from_path, to_path)

def get_library_extension(platform):
    """Return the shared library extension for the given platform."""
    extensions = {
        "windows": ".dll",
        "linux": ".so",
        "macos": ".dylib",
        "android": ".so",
        "ios": ".dylib",
        "web": ".wasm"
    }
    return extensions.get(platform, ".so")

def find_and_copy_external_libs(env, target_dir, platform):
    """Find and copy external libraries from thirdparty/<library>/<platform>/ to target directory."""
    if not external_libs:
        return []
    
    if not os.path.exists("thirdparty"):
        print("Warning: thirdparty directory not found. Skipping external library copy.")
        return []
    
    lib_ext = get_library_extension(platform)
    copied_files = []
    
    for lib_name in external_libs:
        lib_found = False
        
        # Look in thirdparty/<library>/<platform>/
        lib_platform_dir = os.path.join("thirdparty", lib_name, platform)
        
        if os.path.exists(lib_platform_dir):
            # Find all files with the library extension in this directory
            pattern = os.path.join(lib_platform_dir, f"*{lib_ext}*")
            matches = glob.glob(pattern)
            
            if matches:
                for match in matches:
                    dest = os.path.join(target_dir, os.path.basename(match))
                    shutil.copy2(match, dest)
                    copied_files.append(dest)
                    print(f"Copied {match} -> {dest}")
                    lib_found = True
        
        # Fallback: try thirdparty/<library>/ (no platform subfolder)
        if not lib_found:
            lib_generic_dir = os.path.join("thirdparty", lib_name)
            if os.path.exists(lib_generic_dir):
                pattern = os.path.join(lib_generic_dir, f"*{lib_ext}*")
                matches = glob.glob(pattern)
                
                if matches:
                    for match in matches:
                        dest = os.path.join(target_dir, os.path.basename(match))
                        shutil.copy2(match, dest)
                        copied_files.append(dest)
                        print(f"Copied {match} -> {dest}")
                        lib_found = True
        
        if not lib_found:
            print(f"Warning: Could not find {lib_name}{lib_ext} in thirdparty/{lib_name}/{platform}/ or thirdparty/{lib_name}/")
    
    return copied_files

def get_library_paths(env, platform):
    """Get library search paths for linking."""
    if not external_libs or not os.path.exists("thirdparty"):
        return []
    
    paths = []
    for lib_name in external_libs:
        lib_platform_dir = os.path.join("thirdparty", lib_name, platform)
        lib_generic_dir = os.path.join("thirdparty", lib_name)
        
        if os.path.exists(lib_platform_dir):
            paths.append(lib_platform_dir)
        elif os.path.exists(lib_generic_dir):
            paths.append(lib_generic_dir)
    
    return paths

customs = ["custom.py"]
customs = [os.path.abspath(path) for path in customs]

opts = Variables(customs, ARGUMENTS)
opts.Update(localEnv)

Help(opts.GenerateHelpText(localEnv))

env = localEnv.Clone()

if not (os.path.isdir("godot-cpp") and os.listdir("godot-cpp")):
    print_error("""godot-cpp is not available within this folder, as Git submodules haven't been initialized.
Run the following command to download godot-cpp:

    git submodule update --init --recursive""")
    sys.exit(1)

env = SConscript("godot-cpp/SConstruct", {"env": env, "customs": customs})

env.Append(CPPPATH=["src/"])

# Add library paths for each external library
lib_paths = get_library_paths(env, env['platform'])
if lib_paths:
    env.Append(LIBPATH=lib_paths)
    env.Append(LIBS=external_libs)

sources = Glob("src/*.cpp")

if env["target"] in ["editor", "template_debug"]:
    try:
        doc_data = env.GodotCPPDocData("src/gen/doc_data.gen.cpp", source=Glob("doc_classes/*.xml"))
        sources.append(doc_data)
    except AttributeError:
        print("Not including class reference as we're targeting a pre-4.3 baseline.")

# .dev doesn't inhibit compatibility, so we don't need to key it.
# .universal just means "compatible with all relevant arches" so we don't need to key it.
suffix = env['suffix'].replace(".dev", "").replace(".universal", "")

lib_filename = "{}{}{}{}".format(env.subst('$SHLIBPREFIX'), libname, suffix, env.subst('$SHLIBSUFFIX'))

# Build to demo/addons/example_plugin/bin
addon_bin_dir = "{}/addons/{}/bin/{}".format(projectdir, addon_name, env['platform'])

library = env.SharedLibrary(
    "{}/{}".format(addon_bin_dir, lib_filename),
    source=sources,
)

# Copy external libraries after build
def copy_external_libs_action(target, source, env):
    target_dir = os.path.dirname(str(target[0]))
    find_and_copy_external_libs(env, target_dir, env['platform'])
    return None

# Add a post-build action to copy external libraries
copy_libs_cmd = env.Command(
    "{}/._external_libs_copied".format(addon_bin_dir),
    library,
    copy_external_libs_action
)

default_args = [library, copy_libs_cmd]
Default(*default_args)

# Copyright © 2017 Intel Corporation

# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

tu_entrypoints = custom_target(
  'tu_entrypoints',
  input : [vk_entrypoints_gen, vk_api_xml],
  output : ['tu_entrypoints.h', 'tu_entrypoints.c'],
  command : [
    prog_python, '@INPUT0@', '--xml', '@INPUT1@', '--proto', '--weak',
    '--out-h', '@OUTPUT0@', '--out-c', '@OUTPUT1@', '--prefix', 'tu',
  ],
  depend_files : vk_entrypoints_gen_depend_files,
)


libtu_files = files(
  'tu_autotune.c',
  'tu_clear_blit.c',
  'tu_cmd_buffer.c',
  'tu_cs.c',
  'tu_cs.h',
  'tu_device.c',
  'tu_descriptor_set.c',
  'tu_descriptor_set.h',
  'tu_formats.c',
  'tu_image.c',
  'tu_nir_lower_multiview.c',
  'tu_pass.c',
  'tu_pipeline.c',
  'tu_pipeline_cache.c',
  'tu_private.h',
  'tu_query.c',
  'tu_shader.c',
  'tu_util.c',
  'tu_util.h',
  'tu_perfetto.h',
  'vk_format.h',
)

tu_deps = []
tu_flags = []

tu_wsi = false

if with_platform_x11
  tu_deps += dep_xcb_dri3
  tu_wsi = true
endif

if with_platform_wayland
  tu_deps += dep_wayland_client
  tu_wsi = true
endif

if system_has_kms_drm and not with_platform_android
  tu_wsi = true
endif

if tu_wsi
  libtu_files += 'tu_wsi.c'
endif

if with_platform_android
  libtu_files += files('tu_android.c')
  tu_deps += [dep_android]
endif

if with_xlib_lease
  tu_deps += [dep_xlib_xrandr]
endif

if with_freedreno_kgsl
  tu_flags += '-DTU_USE_KGSL'
  libtu_files += files('tu_kgsl.c')
else
  libtu_files += files('tu_drm.c')
  tu_deps += dep_libdrm
endif

tu_tracepoints = custom_target(
  'tu_tracepoints.[ch]',
  input: 'tu_tracepoints.py',
  output: ['tu_tracepoints.c', 'tu_tracepoints.h', 'tu_tracepoints_perfetto.h'],
  command: [
    prog_python, '@INPUT@',
    '-p', join_paths(meson.source_root(), 'src/util/perf/'),
    '--utrace-src', '@OUTPUT0@',
    '--utrace-hdr', '@OUTPUT1@',
    '--perfetto-hdr', '@OUTPUT2@',
  ],
  depend_files: u_trace_py,
)

if with_perfetto
  libtu_files += ['tu_perfetto.cc', 'tu_perfetto_util.c']
  tu_deps += dep_perfetto
endif

libvulkan_freedreno = shared_library(
  'vulkan_freedreno',
  [libtu_files, tu_entrypoints, tu_tracepoints, freedreno_xml_header_files, sha1_h],
  include_directories : [
    inc_include,
    inc_src,
    inc_mapi,
    inc_mesa,
    inc_gallium,
    inc_gallium_aux,
    inc_compiler,
    inc_freedreno,
  ],
  link_with : [
    libfreedreno_ir3,
    libfreedreno_layout,
    libfreedreno_perfcntrs,
  ],
  dependencies : [
    idep_libfreedreno_common,
    dep_dl,
    dep_elf,
    dep_m,
    dep_thread,
    dep_valgrind,
    idep_nir,
    tu_deps,
    idep_vulkan_util,
    idep_vulkan_runtime,
    idep_vulkan_wsi,
    idep_mesautil,
  ],
  c_args : [no_override_init_args, tu_flags],
  gnu_symbol_visibility : 'hidden',
  link_args : [ld_args_bsymbolic, ld_args_gc_sections, ld_args_build_id],
  install : true,
)

if with_symbols_check
  test(
    'tu symbols check',
    symbols_check,
    args : [
      '--lib', libvulkan_freedreno,
      '--symbols-file', vulkan_icd_symbols,
      symbols_check_args,
    ],
    suite : ['freedreno'],
  )
endif

freedreno_icd = custom_target(
  'freedreno_icd',
  input : [vk_icd_gen, vk_api_xml],
  output : 'freedreno_icd.@0@.json'.format(host_machine.cpu()),
  command : [
    prog_python, '@INPUT0@',
    '--api-version', '1.1', '--xml', '@INPUT1@',
    '--lib-path', join_paths(get_option('prefix'), get_option('libdir'),
                             'libvulkan_freedreno.so'),
    '--out', '@OUTPUT@',
  ],
  build_by_default : true,
  install_dir : with_vulkan_icd_dir,
  install : true,
)

if meson.version().version_compare('>= 0.58')
  _dev_icdname = 'freedreno_devenv_icd.@0@.json'.format(host_machine.cpu())
  custom_target(
    'freedreno_devenv_icd',
    input : [vk_icd_gen, vk_api_xml],
    output : _dev_icdname,
    command : [
      prog_python, '@INPUT0@',
      '--api-version', '1.1', '--xml', '@INPUT1@',
      '--lib-path', meson.current_build_dir() / 'libvulkan_freedreno.so',
      '--out', '@OUTPUT@',
    ],
    build_by_default : true,
  )

  devenv.append('VK_ICD_FILENAMES', meson.current_build_dir() / _dev_icdname)
endif

# SPDX-License-Identifier: LGPL-2.1-or-later

systemd_journal_upload_sources = files(
        'journal-upload-journal.c',
        'journal-upload.c',
)

libsystemd_journal_remote_sources = files(
        'journal-remote-parse.c',
        'journal-remote-write.c',
        'journal-remote.c',
)

if conf.get('HAVE_MICROHTTPD') == 1
        libsystemd_journal_remote_sources += files(
                'microhttpd-util.c',
        )
endif

libsystemd_journal_remote = static_library(
        'systemd-journal-remote',
        libsystemd_journal_remote_sources,
        include_directories : journal_includes,
        link_with : libjournal_core,
        dependencies : [libgnutls,
                        liblz4,
                        libmicrohttpd,
                        libxz,
                        threads,
                        userspace],
        build_by_default : false)

systemd_journal_remote_sources = files('journal-remote-main.c')

systemd_journal_gatewayd_sources = files(
        'journal-gatewayd.c',
        'microhttpd-util.c',
)

in_files = [
        ['journal-upload.conf',
         conf.get('ENABLE_REMOTE') == 1 and conf.get('HAVE_LIBCURL') == 1 and install_sysconfdir_samples],
        ['journal-remote.conf',
         conf.get('ENABLE_REMOTE') == 1 and conf.get('HAVE_MICROHTTPD') == 1 and install_sysconfdir_samples]]

foreach tuple : in_files
        file = tuple[0]
        custom_target(
                file,
                input : file + '.in',
                output: file,
                command : [jinja2_cmdline, '@INPUT@', '@OUTPUT@'],
                install : tuple[1],
                install_dir : pkgsysconfdir)
endforeach

if conf.get('ENABLE_REMOTE') == 1 and conf.get('HAVE_MICROHTTPD') == 1
        install_data('browse.html',
                     install_dir : pkgdatadir / 'gatewayd')

        if get_option('create-log-dirs')
                meson.add_install_script('sh', '-c',
                                         mkdir_p.format('/var/log/journal/remote'))
                meson.add_install_script('sh', '-c',
                                         '''chown 0:0 $DESTDIR/var/log/journal/remote &&
                                            chmod 755 $DESTDIR/var/log/journal/remote || :''')
        endif
endif

############################################################

fuzzers += [
        {
                'sources' : files('fuzz-journal-remote.c'),
                'link_with' : [
                        libshared,
                        libsystemd_journal_remote,
                ],
                'includes' : journal_includes,
        },
]

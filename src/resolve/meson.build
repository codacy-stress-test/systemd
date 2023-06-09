# SPDX-License-Identifier: LGPL-2.1-or-later

resolve_includes = [includes, include_directories('.')]

basic_dns_sources = files(
        'resolved-dns-dnssec.c',
        'resolved-dns-packet.c',
        'resolved-dns-rr.c',
        'resolved-dns-answer.c',
        'resolved-dns-question.c',
        'resolved-util.c',
        'dns-type.c',
)

systemd_resolved_sources = files(
        'resolved-bus.c',
        'resolved-conf.c',
        'resolved-dns-cache.c',
        'resolved-dns-query.c',
        'resolved-dns-scope.c',
        'resolved-dns-search-domain.c',
        'resolved-dns-server.c',
        'resolved-dns-stream.c',
        'resolved-dns-stub.c',
        'resolved-dns-synthesize.c',
        'resolved-dns-transaction.c',
        'resolved-dns-trust-anchor.c',
        'resolved-dns-zone.c',
        'resolved-dnssd-bus.c',
        'resolved-dnssd.c',
        'resolved-etc-hosts.c',
        'resolved-link-bus.c',
        'resolved-link.c',
        'resolved-llmnr.c',
        'resolved-manager.c',
        'resolved-mdns.c',
        'resolved-resolv-conf.c',
        'resolved-socket-graveyard.c',
        'resolved-varlink.c',
)

resolvectl_sources = files(
        'resolvconf-compat.c',
        'resolvectl.c',
)

############################################################

dns_type_list_txt = custom_target(
        'dns_type-list.txt',
        input : ['generate-dns_type-list.sed', 'dns-type.h'],
        output : 'dns_type-list.txt',
        command : [sed, '-n', '-r', '-f', '@INPUT0@', '@INPUT1@'],
        capture : true)

generate_dns_type_gperf = find_program('generate-dns_type-gperf.py')

gperf_file = custom_target(
        'dns_type-from-name.gperf',
        input : dns_type_list_txt,
        output : 'dns_type-from-name.gperf',
        command : [generate_dns_type_gperf, 'dns_type', 'DNS_TYPE_', '@INPUT@'],
        capture : true)

basic_dns_sources += custom_target(
        'dns_type-from-name.h',
        input : gperf_file,
        output : 'dns_type-from-name.h',
        command : [gperf,
                   '-L', 'ANSI-C', '-t', '--ignore-case',
                   '-N', 'lookup_dns_type',
                   '-H', 'hash_dns_type_name',
                   '-p', '-C',
                   '@INPUT@'],
        capture : true)

basic_dns_sources += custom_target(
        'dns_type-to-name.h',
        input : ['dns_type-to-name.awk', dns_type_list_txt],
        output : 'dns_type-to-name.h',
        command : [awk, '-f', '@INPUT0@', '@INPUT1@'],
        capture : true)

libsystemd_resolve_core = static_library(
        'systemd-resolve-core',
        basic_dns_sources,
        include_directories : includes,
        dependencies : userspace,
        build_by_default : false)

systemd_resolved_sources += custom_target(
        'resolved_gperf.c',
        input : 'resolved-gperf.gperf',
        output : 'resolved-gperf.c',
        command : [gperf, '@INPUT@', '--output-file', '@OUTPUT@'])

systemd_resolved_sources += custom_target(
        'resolved_dnssd_gperf.c',
        input : 'resolved-dnssd-gperf.gperf',
        output : 'resolved-dnssd-gperf.c',
        command : [gperf, '@INPUT@', '--output-file', '@OUTPUT@'])

systemd_resolved_dependencies = [threads, libm] + [lib_openssl_or_gcrypt]
if conf.get('ENABLE_DNS_OVER_TLS') == 1
        if conf.get('DNS_OVER_TLS_USE_GNUTLS') == 1
                systemd_resolved_sources += files(
                        'resolved-dnstls-gnutls.c',
                )
                systemd_resolved_dependencies += libgnutls
        elif conf.get('DNS_OVER_TLS_USE_OPENSSL') == 1
                systemd_resolved_sources += files(
                        'resolved-dnstls-openssl.c',
                )
                systemd_resolved_dependencies += libopenssl
        else
                error('unknown dependency for supporting DNS-over-TLS')
        endif
endif

if conf.get('ENABLE_RESOLVE') == 1
        install_data('org.freedesktop.resolve1.conf',
                     install_dir : dbuspolicydir)
        install_data('org.freedesktop.resolve1.service',
                     install_dir : dbussystemservicedir)
        install_data('org.freedesktop.resolve1.policy',
                     install_dir : polkitpolicydir)
        install_data('resolv.conf',
                     install_dir : rootlibexecdir)
endif

custom_target(
        'resolved.conf',
        input : 'resolved.conf.in',
        output : 'resolved.conf',
        command : [jinja2_cmdline, '@INPUT@', '@OUTPUT@'],
        install : conf.get('ENABLE_RESOLVE') == 1 and install_sysconfdir_samples,
        install_dir : pkgsysconfdir)

############################################################

test_resolve_base = {
        'link_with' : [libsystemd_resolve_core, libshared],
        'dependencies' : [lib_openssl_or_gcrypt, libm],
}

tests += [
        {
                'sources' : files('test-resolve-tables.c'),
                'base' : test_resolve_base,
        },
        {
                'sources' : files('test-dns-packet.c'),
                'base' : test_resolve_base,
        },
        {
                'sources' : files(
                        'test-resolved-etc-hosts.c',
                        'resolved-etc-hosts.c',
                ),
                'base' : test_resolve_base,
        },
        {
                'sources' : files('test-resolved-packet.c'),
                'base' : test_resolve_base,
        },
        {
                'sources' : files('test-dnssec.c'),
                'base' : test_resolve_base,
                'condition' : 'HAVE_OPENSSL_OR_GCRYPT',
        },
        {
                'sources' : files('test-dnssec-complex.c'),
                'base' : test_resolve_base,
                'type' : 'manual',
        },
        {
                'sources' : [
                        files('test-resolved-stream.c'),
                        basic_dns_sources,
                        systemd_resolved_sources,
                ],
                'dependencies' : [
                        lib_openssl_or_gcrypt,
                        libm,
                        systemd_resolved_dependencies,
                ],
                'includes' : resolve_includes,
        },
]

fuzzer_resolve_base = {
        'link_with' : [libsystemd_resolve_core, libshared],
        'dependencies' : [lib_openssl_or_gcrypt, libm],
}

fuzzers += [
        {
                'sources' : files('fuzz-dns-packet.c'),
                'base' : fuzzer_resolve_base,
        },
        {
                'sources' : files(
                        'fuzz-etc-hosts.c',
                        'resolved-etc-hosts.c',
                ),
                'base' : fuzzer_resolve_base,
        },
        {
                'sources' : files('fuzz-resource-record.c'),
                'base' : fuzzer_resolve_base,
        },
]

systemd_resolved_sources += files('resolved.c')

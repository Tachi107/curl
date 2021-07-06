project(
	'curl',
	'c',
	version: '7.78.0',
	default_options: [
		'b_ndebug=if-release'
	],
	meson_version: '>=0.51.0' # 0.50 con has_headers in find_library, 0.51 con get_variable(type: 'variable')
)

# Credo che tutte le variabili siano esportate in CMake per configurare il file curl_config.h.cmake,
# che credo vada poi a definire tutte le varie funzionailtà di curl nei vari #define.
# lib/CMakeLists.txt#L33.
# Questo perché curl non decide cose in base al sistema operativo, ma in base alle funzionalità disponibili;
# quindi devo controllare se ogni singola funzionalità è supportata, e se lo è la metto in un array.
# Anzi, al posto dell'array credo che debba usare un oggetto configuration_data: https://mesonbuild.com/Configuration.html
# Posso fare tipo conf_data.set('HAVE_THAT_WEIRD_FUNCTION', cc.have_function('weird_function'))
# cc.has_header_symbol() per controllare se un header ha un determinato simbolo
#
# TODO:
#   I nomi delle opzioni sono inconsistenti
#   Ordinare alfabeticamente i vari check cc.has_header(), cc.sizeof(), ecc

conf = configuration_data()
cc = meson.get_compiler('c')
os = host_machine.system()
curl_deps = []

conf.set_quoted('OS', os)

if os == 'windows'
	add_project_arguments('-D_WIN32_WINNT=' + get_option('curl_target_windows_version'), language: 'c')
	if get_option('enable_unicode')                      # -municode is only for MinGW
		add_project_arguments('-DUNICODE', '-D_UNICODE', cc.get_supported_arguments('-municode'), language: 'c')
	endif
endif

if get_option('debug')
	add_project_arguments('-DDEBUGBUILD', '-DCURLDEBUG', language: 'c')
endif
if get_option('enable_curldebug')
	add_project_arguments('-DCURLDEBUG', language: 'c')
endif

if get_option('enable_ares')
	curl_deps += dependency('libcares')
	conf.set('USE_ARES', true)
	if get_option('enable_threaded_resolver')
		threads_dep = dependency('threads')
		curl_deps += threads_dep
		if os == 'windows'
			conf.set('USE_THREADS_WIN32', true)
		else
			conf.set('USE_THREADS_POSIX', cc.has_function('pthread_create', dependencies: threads_dep))
			conf.set('HAVE_PTHREAD_H', cc.has_header('pthread.h', dependencies: threads_dep))
		endif
	endif
endif

if get_option('enable_ipv6')
	conf.set('ENABLE_IPV6', true)
	if os != 'windows'
		conf.set('HAVE_SOCKADDR_IN6_SIN6_SCOPE_ID', cc.has_member('struct sockaddr_in6', 'sin6_scope_id', prefix: '#include <netinet/in.h>'))
		if not cc.has_member('struct sockaddr_in6', 'sin6_addr', prefix: '#include <netinet/in.h>')
			warning('struct sockaddr_in6 not available, disabling IPv6 support')
			conf.set('ENABLE_IPV6', false)
		endif
	endif
endif

# check_library_exists_concat prende la libreria in cui cercare,
# il simbolo che dev'essere nella libreria, e una variabile in cui
# conservare il risultato del controllo.
# Se la libreria contiene il simbolo cercato allora questa viene
# aggiunta alla lista delle librerie richieste da curl

# Check if -ldl is required to use dlopen
if not cc.has_function('dlopen')
	dl_dep = cc.find_library('dl')
	if cc.has_function('dlopen', dependencies: dl_dep)
		curl_deps += dl_dep
	endif
endif

if not cc.has_function('connect')
	socket_dep = cc.find_library('socket')
	if cc.has_function('connect', dependencies: socket_dep)
		curl_deps += socket_dep
		conf.set('HAVE_LIBSOCKET', true)
	endif
endif

if not cc.has_function('gethostbyname')
	c_dep = cc.find_library('c', required: false)
	nsl_dep = cc.find_library('nsl', required: false)
	if cc.has_function('gethostbyname', dependencies: c_dep)
		curl_deps += c_dep
	elif cc.has_function('gethostbyname', dependencies: nsl_dep)
		curl_deps += nsl_dep
	endif
endif

conf.set('HAVE_GETHOSTNAME', cc.has_function('gethostname'))

if os == 'windows'
	if not cc.has_function('getch')
		ws2_32_dep = cc.find_library('ws2_32', required: false)
		winmm_dep = cc.find_library('winmm', required: false)
		if cc.has_function('getch', dependencies: ws2_32_dep)
			curl_deps += ws2_32_dep
		elif cc.has_function('getch', dependencies: winmm_dep)
			curl_deps += winmm_dep
		endif
	endif
endif

# Check SSL/TLS libraries
ssl_backends = get_option('ssl_backend')
curl_with_multi_ssl = false
if ssl_backends.length() > 1
	curl_with_multi_ssl = true
endif

if ssl_backends.contains('openssl')
	openssl_dep = dependency('openssl')
	curl_deps += openssl_dep
	conf.set('USE_OPENSSL', true)
	conf.set('HAVE_OPENSSL_CRYPTO_H', cc.has_header('openssl/crypto.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_ERR_H',    cc.has_header('openssl/err.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_PEM_H',    cc.has_header('openssl/pem.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_RSA_H',    cc.has_header('openssl/rsa.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_SSL_H',    cc.has_header('openssl/ssl.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_X509_H',   cc.has_header('openssl/x509.h', dependencies: openssl_dep))
	conf.set('HAVE_OPENSSL_RAND_H',   cc.has_header('openssl/rand.h', dependencies: openssl_dep))
	conf.set('HAVE_RAND_STATUS', cc.has_function('RAND_status', dependencies: openssl_dep))
	conf.set('HAVE_RAND_SCREEN', cc.has_function('RAND_screen', dependencies: openssl_dep))
	conf.set('HAVE_RAND_EGD',    cc.has_function('RAND_egd', dependencies: openssl_dep))
	if get_option('curl_disable_openssl_auto_load_config')
		add_project_arguments('-DCURL_DISABLE_OPENSSL_AUTO_LOAD_CONFIG', language: 'c')
	endif
endif

if ssl_backends.contains('wolfssl')
	curl_deps += dependency('wolfssl')
	conf.set('USE_WOLFSSL', true)
endif

if ssl_backends.contains('gnutls')
	curl_deps += dependency('gnutls')
	conf.set('USE_GNUTLS', true)
endif

if ssl_backends.contains('nss')
	nss_dep = dependency('nss')
	curl_deps += nss_dep
	conf.set('USE_NSS', true)
	conf.set('HAVE_PK11_CREATEMANAGEDGENERICOBJECT', cc.has_function('PK11_CreateManagedGenericObject', dependencies: nss_dep))
endif

if ssl_backends.contains('mbedtls')
	mbedtls_dep = dependency('mbedtls', required: false) # unavailable until ARMmbed/mbedtls#228 is fixed
	if mbedtls_dep.found()
		curl_deps += mbedtls_dep
	else
		curl_deps += [cc.find_library('mbedtls', has_headers: 'mbedtls/ssl.h'), cc.find_library('mbedx509'), cc.find_library('mbedcrypto')]		
	endif
	conf.set('USE_MBEDTLS', true)
endif

if ssl_backends.contains('bearssl')
	curl_deps += dependency('bearssl') # can't test
	conf.set('USE_BEARSSL', true)
endif

if ssl_backends.contains('schannel')
	conf.set('USE_SCHANNEL', true)
	conf.set('USE_WINDOWS_SSPI', true)
	if get_option('curl_windows_sspi')
		add_project_arguments('-DSECURITY_WIN32', language: 'c')
	endif
endif

if os == 'darwin'
	curl_deps += [cc.find_library('CoreFoundation'), cc.find_library('SystemConfiguration')]
	if ssl_backends.contains('sectransp')
		curl_deps += cc.find_library('Security')
		conf.set('USE_SECTRANSP', true)
	endif
endif

if get_option('use_nghttp2')
	curl_deps += dependency('libnghttp2')
	conf.set('USE_NGHTTP2', true)
endif

if get_option('http3_backend') == 'ngtcp2' # can't check
	if ssl_backends.contains('openssl')
		if not cc.has_function('SSL_CTX_set_quic_method', dependencies: dependency('openssl'))
			error('QUIC support is missing in OpenSSL/BoringSSL')
		endif
		curl_deps += dependency('ngtcp2', components: 'openssl')
	elif ssl_backends.contains('gnutls')
		curl_deps += dependency('ngtcp2', components: 'gnutls')
	endif
	conf.set('USE_NGTCP2', true)
	curl_deps += dependency('nghttp3')
	conf.set('USE_NGHTTP3', true)
elif get_option('http3_backend') == 'quiche' # can't check
	if not cc.has_function('SSL_CTX_set_quic_method', dependencies: dependency('openssl'))
		error('QUIC support is missing in OpenSSL/BoringSSL')
	endif
	quiche_dep = dependency('quiche')
	curl_deps += quiche_dep # can't check
	conf.set('USE_QUICHE', true)
	conf.set('HAVE_QUICHE_CONN_SET_QLOG_FD', cc.has_function('quiche_conn_set_qlog_fd', dependencies: quiche_dep))
endif

# controllare autotools, il fatto che non fallisca se la libreria non viene
# trovata non mi convince
if get_option('use_libidn2')
	libidn2_dep = dependency('libidn2', required: false)
	if libidn2_dep.found()
		curl_deps += libidn2_dep
		conf.set('HAVE_LIBIDN2', true)
	endif
endif

if os == 'windows' and get_option('use_win32_idn')
	curl_deps += cc.find_library('Normaliz')
	conf.set('WANT_IDN_PROTOTYPES', true)
endif

zlib_dep = dependency('zlib', required: false)
if zlib_dep.found()
	conf.set('HAVE_LIBZ', true)
	conf.set('HAVE_ZLIB_H', true)
endif

if get_option('curl_brotli')
	curl_deps += [dependency('libbrotlicommon'), dependency('libbrotlidec')]
	conf.set('HAVE_BROTLI', true)
endif

if get_option('curl_zstd')
	zstd_dep = dependency('libzstd')
	if cc.has_function('ZSTD_createDStream', dependencies: zstd_dep)
		curl_deps += zstd_dep
		conf.set('HAVE_ZSTD', true)
	endif
endif

have_libssh2 = false
if get_option('use_libssh2')
	libssh2_dep = dependency('libssh2', required: false)
	if libssh2_dep.found()
		curl_deps += libssh2_dep
		have_libssh2 = true
		conf.set('USE_LIBSSH2', true)
		conf.set('HAVE_LIBSSH2', true)
		conf.set('HAVE_LIBSSH2_H', true)
	endif
endif

if get_option('use_libssh')
	if not have_libssh2
		curl_deps += dependency('libssh')
		conf.set('USE_LIBSSH', true)
		conf.set('HAVE_LIBSSH_LIBSSH_H', true)
	else
		warning('You enabled both libssh2 and libssh support, using only libssh2')
	endif
endif

if get_option('use_gssapi')
	gssapi_dep = dependency('krb5-gssapi')
	curl_deps += gssapi_dep
	conf.set('HAVE_GSSAPI', true)
	conf.set('HAVE_GSSAPI_GSSAPI_H',         cc.has_header('gssapi/gssapi.h'))
	conf.set('HAVE_GSSAPI_GSSAPI_GENERIC_H', cc.has_header('gssapi/gssapi_generic.h'))
	conf.set('HAVE_GSSAPI_GSSAPI_KRB5_H',    cc.has_header('gssapi/gssapi_krb5.h'))
	gssapi_vendor = gssapi_dep.get_variable(pkgconfig: 'vendor')
	if gssapi_vendor == 'Heimdal'
		conf.set('HAVE_GSSHEIMDAL', true)
	elif gssapi_vendor == 'MIT'
		conf.set('HAVE_GSSMIT', true)
		conf.set('HAVE_OLD_GSSMIT', not cc.has_header_symbol('gssapi/gssapi.h', 'GSS_C_NT_HOSTBASED_SERVICE', dependencies: gssapi_dep))
	endif
endif

if get_option('enable_unix_sockets')
	if os == 'windows'
		conf.set('USE_UNIX_SOCKETS', true)
	else
		conf.set('USE_UNIX_SOCKETS', cc.has_member('struct sockaddr_un', 'sun_path', prefix: '#include <sys/un.h>'))
	endif
endif

if get_option('curl_ca_bundle') == 'auto'
	fs = import('fs')
	foreach ca_bundle_path : ['/etc/ssl/certs/ca-certificates.crt', '/etc/pki/tls/certs/ca-bundle.crt', '/usr/share/ssl/certs/ca-bundle.crt', '/usr/local/share/certs/ca-root-nss.crt', '/etc/ssl/cert.pem']
		if fs.exists(ca_bundle_path)
			conf.set_quoted('CURL_CA_BUNDLE', ca_bundle_path)
			break
		endif
	endforeach
elif get_option('curl_ca_bundle') == 'none'
	# don't set anything
elif get_option('curl_ca_bundle') == ''
	error('Invalid curl_ca_bundle value (empty)')
else
	conf.set_quoted('CURL_CA_BUNDLE', get_option('curl_ca_bundle'))
endif

ca_path_set = false
if get_option('curl_ca_path') == 'auto' and not ssl_backends.contains('nss')
	if import('fs').exists('/etc/ssl/certs')
		conf.set_quoted('CURL_CA_PATH', '/etc/ssl/certs')
		ca_path_set = true
	endif
elif get_option('curl_ca_path') == 'none'
	# don't set anything
elif get_option('curl_ca_path') == ''
	error('Invalid curl_ca_path value (empty)')
else
	conf.set_quoted('CURL_CA_PATH', get_option('curl_ca_path'))
	ca_path_set = true
endif

if ca_path_set and (not ssl_backends.contains('openssl') and not ssl_backends.contains('gnutls') and not ssl_backends.contains('mbedtls'))
	error('CA path only supported by OpenSSL, GnuTLS or mbedTLS. Set curl_ca_path=none or enable one of those TLS backends.')
endif

conf.set('HAVE_WINDOWS_H', cc.has_header('windows.h'))
conf.set('HAVE_WINSOCK_H', cc.has_header('winsock.h'))
conf.set('HAVE_WINSOCK2_H', cc.has_header('winsock2.h'))
conf.set('HAVE_WS2TCPIP_H', cc.has_header('ws2tcpip.h'))
conf.set('HAVE_WINCRYPT_H', cc.has_header('wincrypt.h'))

conf.set('HAVE_INTTYPES_H', cc.has_header('inttypes.h'))
conf.set('HAVE_SYS_FILIO_H', cc.has_header('sys/filio.h'))
conf.set('HAVE_SYS_IOCTL_H', cc.has_header('sys/ioctl.h'))
conf.set('HAVE_SYS_PARAM_H', cc.has_header('sys/param.h'))
conf.set('HAVE_SYS_POLL_H', cc.has_header('sys/poll.h'))
conf.set('HAVE_SYS_RESOURCE_H', cc.has_header('sys/resource.h'))
conf.set('HAVE_SYS_SELECT_H', cc.has_header('sys/select.h'))
conf.set('HAVE_SYS_SOCKET_H', cc.has_header('sys/socket.h'))
conf.set('HAVE_SYS_SOCKIO_H', cc.has_header('sys/sockio.h'))
conf.set('HAVE_SYS_STAT_H', cc.has_header('sys/stat.h'))
conf.set('HAVE_SYS_TIME_H', cc.has_header('sys/time.h'))
conf.set('HAVE_SYS_TYPES_H', cc.has_header('sys/types.h'))
conf.set('HAVE_SYS_UIO_H', cc.has_header('sys/uio.h'))
conf.set('HAVE_SYS_UN_H', cc.has_header('sys/un.h'))
conf.set('HAVE_SYS_UTIME_H', cc.has_header('sys/utime.h'))
conf.set('HAVE_SYS_XATTR_H', cc.has_header('sys/xattr.h'))
conf.set('HAVE_ALLOCA_H', cc.has_header('alloca.h'))
conf.set('HAVE_ARPA_INET_H', cc.has_header('arpa/inet.h'))
conf.set('HAVE_ARPA_TFTP_H', cc.has_header('arpa/tftp.h'))
conf.set('HAVE_ASSERT_H', cc.has_header('assert.h'))
conf.set('HAVE_ERRNO_H', cc.has_header('errno.h'))
conf.set('HAVE_FCNTL_H', cc.has_header('fcntl.h'))
conf.set('HAVE_IDN2_H', cc.has_header('idn2.h'))
conf.set('HAVE_IFADDRS_H', cc.has_header('ifaddrs.h'))
conf.set('HAVE_IO_H', cc.has_header('io.h'))
conf.set('HAVE_KRB_H', cc.has_header('krb.h'))
conf.set('HAVE_LIBGEN_H', cc.has_header('libgen.h'))
conf.set('HAVE_LOCALE_H', cc.has_header('locale.h'))
conf.set('HAVE_NET_IF_H', cc.has_header('net/if.h'))
conf.set('HAVE_NETDB_H', cc.has_header('netdb.h'))
conf.set('HAVE_NETINET_IN_H', cc.has_header('netinet/in.h'))
conf.set('HAVE_NETINET_TCP_H', cc.has_header('netinet/tcp.h'))
conf.set('HAVE_LINUX_TCP_H', cc.has_header('linux/tcp.h'))

conf.set('HAVE_PEM_H', cc.has_header('pem.h'))
conf.set('HAVE_POLL_H', cc.has_header('poll.h'))
conf.set('HAVE_PWD_H', cc.has_header('pwd.h'))
conf.set('HAVE_SETJMP_H', cc.has_header('setjmp.h'))
conf.set('HAVE_SIGNAL_H', cc.has_header('signal.h'))
conf.set('HAVE_SSL_H', cc.has_header('ssl.h'))
conf.set('HAVE_STDBOOL_H', cc.has_header('stdbool.h'))
conf.set('HAVE_STDINT_H', cc.has_header('stdint.h'))
conf.set('HAVE_STDIO_H', cc.has_header('stdio.h'))
conf.set('HAVE_STDLIB_H', cc.has_header('stdlib.h'))
conf.set('HAVE_STRING_H', cc.has_header('string.h'))
conf.set('HAVE_STRINGS_H', cc.has_header('strings.h'))
conf.set('HAVE_STROPTS_H', cc.has_header('stropts.h'))
conf.set('HAVE_TERMIO_H', cc.has_header('termio.h'))
conf.set('HAVE_TERMIOS_H', cc.has_header('termios.h'))
conf.set('HAVE_TIME_H', cc.has_header('time.h'))
conf.set('HAVE_UNISTD_H', cc.has_header('unistd.h'))
conf.set('HAVE_UTIME_H', cc.has_header('utime.h'))
conf.set('HAVE_X509_H', cc.has_header('x509.h'))

conf.set('HAVE_PROCESS_H', cc.has_header('process.h'))
conf.set('HAVE_STDDEF_H', cc.has_header('stddef.h'))
conf.set('HAVE_DLFCN_H', cc.has_header('dlfcn.h'))
conf.set('HAVE_MALLOC_H', cc.has_header('malloc.h'))
conf.set('HAVE_MEMORY_H', cc.has_header('memory.h'))
conf.set('HAVE_NETINET_IF_ETHER_H', cc.has_header('netinet/if_ether.h'))
conf.set('HAVE_SOCKIO_H', cc.has_header('sockio.h'))
conf.set('HAVE_SYS_UTSNAME_H', cc.has_header('sys/utsname.h'))

conf.set('SIZEOF_SIZE_T', cc.sizeof('size_t'))
conf.set('SIZEOF_SSIZE_T', cc.sizeof('ssize_t'))
conf.set('SIZEOF_LONG', cc.sizeof('long'))
conf.set('SIZEOF_LONG_LONG', cc.sizeof('long long'))
conf.set('SIZEOF_INT', cc.sizeof('int'))
conf.set('SIZEOF___INT64', cc.sizeof('__int64'))
conf.set('SIZEOF_LONG_DOUBLE', cc.sizeof('long double'))
conf.set('SIZEOF_TIME_T', cc.sizeof('time_t'))

configure_file(output: 'paolone.h', configuration: conf)

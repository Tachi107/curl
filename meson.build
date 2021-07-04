project(
	'curl',
	'c',
	version: '7.78.0',
	default_options: [
		'b_ndebug=if-release'
	],
	meson_version: '>=0.50.0' # 0.50 con has_headers in find_library
)

# Credo che tutte le variabili siano esportate in CMake per configurare il file curl_config.h.cmake,
# che credo vada poi a definire tutte le varie funzionailtà di curl nei vari #define.
# lib/CMakeLists.txt#L33.
# Questo perché curl non decide cose in base al sistema operativo, ma in base alle funzionalità disponibili;
# quindi devo controllare se ogni singola funzionalità è supportata, e se lo è la metto in un array.
# Anzi, al posto dell'array credo che debba usare un oggetto configuration_data: https://mesonbuild.com/Configuration.html
# Posso fare tipo conf_data.set('HAVE_THAT_WEIRD_FUNCTION', cc.have_function('weird_function'))
# cc.has_header_symbol() per controllare se un header ha un determinato simbolo

conf = configuration_data()
cc = meson.get_compiler('c')
os = host_machine.system()
curl_deps = []

conf.set_quoted('OS', os)

if os == 'windows'
	add_project_arguments('-D_WIN32_WINNT=' + get_option('curl_target_windows_version'), language: 'c')
	if get_option('enable_unicode')                      # -municode is only for MinGW
		add_project_arguments('-DUNICODE', '-D_UNICODE', cc.get_supported_arguments('-municode'))
	endif
endif

if get_option('debug')
	add_project_arguments('-DDEBUGBUILD', '-DCURLDEBUG')
endif
if get_option('enable_curldebug')
	add_project_arguments('-DCURLDEBUG')
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
		add_project_arguments('-DCURL_DISABLE_OPENSSL_AUTO_LOAD_CONFIG')
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
		add_project_arguments('-DSECURITY_WIN32')
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

configure_file(output: 'paolone.h', configuration: conf)

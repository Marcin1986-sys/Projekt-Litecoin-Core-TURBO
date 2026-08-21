Raport z Przebudowy Kodu
Litecoin Core v0.21.5.6 – Migracja do CMake i C++20

Niniejszy dokument stanowi zredagowane streszczenie przeprowadzonych
prac programistycznych nad przebudową programu Litecoin Core (wersja
0.21.5.6). Dokument zawiera wyłącznie pomyślnie zweryfikowane kroki,
odfiltrowane ze ślepych zaułków i prób, w celu stworzenia czytelnego opisu
modernizacji systemu budowania do CMake oraz standardu kompilacji C+
+20.

1. Nowa Architektura CMakeLists.txt
Głównym krokiem było przeniesienie logiki ze starego systemu Autotools.
Kod źródłowy podzielono na moduły budowane jako niezależne biblioteki
statyczne (m m.in. litecoin_crypto , univalue , secp256k1_zkp ,
litecoin_consensus , litecoin_util oraz zewnętrzne zależności jak
wbudowane leveldb ).
Kluczowe ustawienia kompilatora dla CMake:

cmake_minimum_required(VERSION 3.16)
project(LitecoinCore VERSION 0.21.5.6 LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
# Dodanie rygorystycznych flag ostrzeżeń i definicji

add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wno-
deprecated-declarations)

add_definitions(-DHAVE_CONFIG_H -DHAVE_BUILD_INFO -DUSE_LIBEVENT)

2. Obsługa Platformy i Zależności Systemowych
Przejście na nowy kompilator GCC 14 oraz bibliotekę systemową glibc w
Linuksie obnażyło redefinicje wbudowanych makr i funkcji (takich jak
Endianness, funkcje bswap oraz strnlen ). Aby rozwiązać konflikty podczas
linkowania obiektów sieciowych, wdrożono scentralizowany plik
konfiguracyjny.
Rozwiązanie: src/config/bitcoin-config.h
Wymuszono flagi HAVE_DECL_* , które dezaktywują redundatne bloki kodu
w compat.h i compat/endian.h .

#ifndef BITCOIN_CONFIG_H
#define BITCOIN_CONFIG_H
#define PACKAGE_NAME "Litecoin Core"
#define PACKAGE_VERSION "0.21.5.6"
#define COPYRIGHT_YEAR 2026
#define COPYRIGHT_HOLDERS "The %s developers"
#define COPYRIGHT_HOLDERS_SUBSTITUTION "Litecoin Core"
/* Blokada redefinicji funkcji systemowych glibc */
#define HAVE_DECL_STRNLEN 1
#define HAVE_DECL_HTOBE16 1
#define HAVE_DECL_HTOLE16 1
#define HAVE_DECL_BE16TOH 1
#define HAVE_DECL_LE16TOH 1
#define HAVE_DECL_HTOBE32 1
#define HAVE_DECL_BSWAP_16 1
#define HAVE_DECL_BSWAP_32 1
#define HAVE_DECL_BSWAP_64 1
#define HAVE_GMTIME_R 1
#endif // BITCOIN_CONFIG_H

3. Refaktoryzacja Alokatorów pod C++20
Aktualizacja standardu do C++20 spowodowała wycofanie starszych aliasów
typów (typedefów) w szablonie std::allocator<T> . Moduły dbające o
zacieranie pamięci kryptograficznej po użyciu musiały zostać napisane na
nowo.

Modernizacja: zeroafterfree.h oraz secure.h
Usunięto odwołania w postaci base::pointer , wprowadzając bezpośrednie
definicje operujące na wskaźnikach T* .

template <typename T>
struct zero_after_free_allocator : public std::allocator<T> {
typedef std::allocator<T> base;
typedef typename base::size_type size_type;
typedef typename base::difference_type difference_type;
// C++20: Wprost zdefiniowane typy zastępujące odwołania do
usuniętego base::pointer
typedef T* pointer;
typedef const T* const_pointer;
typedef T& reference;
typedef const T& const_reference;
typedef typename base::value_type value_type;
void deallocate(T* p, size_t n) {
if (p != nullptr)
memory_cleanse(p, sizeof(T) * n);
std::allocator<T>::deallocate(p, n);
}
};

4. Obsługa najnowszych wersji Bibliotek: fmt oraz
LevelDB
4.1. Biblioteka fmt v10 w kodzie MimbleWimble (MWEB)
Aktualizacja biblioteki formatującej fmt do wersji 10 (domyślnej na
nowszych dystrybucjach Linux) włączyła obostrzenie bezpieczeństwa dla
funkcji typu consteval . Ciągi formatujące przekazywane w czasie
wykonywania aplikacji nie mogą być już przetwarzane domyślnie. Wymaga
to owinięcia zmiennej formatu klasą fmt::runtime .
Plik modyfikowany: src/libmw/include/mw/util/StringUtil.h

// Kod C++ (C++20 & fmt v10 compatible)
template<typename ... Args>
static std::string Format(const char* format, const Args&...
args) {
return fmt::format(fmt::runtime(format),
ConvertArg(args)...);
}

4.2. Wbudowany silnik LevelDB
Baza danych współdzielona jako wewnętrzny magazyn zależała od
archaicznych flag platformy POSIX. Aby bezkolizyjnie zlinkować moduł
leveldb wygenerowano dedykowany nagłówek portu ucinający
niezgodności porządku bajtów oraz dostosowano flagi atomowe do nowego
standardu std.
Modyfikacja 1: port_config.h

#ifndef STORAGE_LEVELDB_PORT_PORT_CONFIG_H_
#define STORAGE_LEVELDB_PORT_PORT_CONFIG_H_
#define LEVELDB_IS_BIG_ENDIAN 0
#define HAVE_FDATASYNC 1
#define HAVE_O_CLOEXEC 1
#endif

Modyfikacja 2: env_posix.cc
Stare atrybuty std::memory_order::memory_order_relaxed , usunięte w
nowych definicjach nagłówkowych C++20, zostały zaktualizowane w
implementacji menedżera pamięci bazy LevelDB do
std::memory_order_relaxed .

5. Kompilacja Demona Klienta (litecoin-cli)
Zwieńczeniem tej iteracji deweloperskiej była integracja podmodułu dla
secp256k1_zkp z aktywowanymi modułami ENABLE_MODULE_EXTRAKEYS=1 ,
ENABLE_MODULE_SCHNORRSIG=1 (wymaganymi do obsługi MWEB i podpisów
Schnorra), a następnie finalna egzekucja kompilacji CMake.

Wynik: Po zaaplikowaniu modyfikacji środowiska polecenie:
cmake --build . --target litecoin-cli -j$(nproc)
zakończyło się powodzeniem, budując biblioteki litecoin_crypto ,
univalue , secp256k1_zkp , litecoin_util i z sukcesem linkując
pierwszy plik binarny: litecoin-cli (Litecoin Core RPC client version
v0.21.5.6).

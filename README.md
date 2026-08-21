
Lista zmodyfikowanych plików w procesie przebudowy:
CMakeLists.txt
src/config/bitcoin-config.h
src/support/allocators/zeroafterfree.h
src/support/allocators/secure.h
src/libmw/include/mw/util/StringUtil.h
src/leveldb/port/port_config.h
src/leveldb/util/env_posix.cc
Przebudowa krok po kroku
Oto szczegółowy opis modyfikacji wprowadzonych w poszczególnych plikach.
1. CMakeLists.txt
Zastąpiliśmy stary system budowania Autotools nowoczesnym silnikiem CMake. Pozwoliło to na podzielenie kodu źródłowego na niezależne moduły (biblioteki statyczne) i wymuszenie standardu kompilacji dla nowoczesnych systemów.
Zastosowany kod (fragment konfiguracyjny CMake):


```bash
CMake
cmake_minimum_required(VERSION 3.16)
project(LitecoinCore VERSION 0.21.5.6 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wno-deprecated-declarations)
add_definitions(-DHAVE_CONFIG_H -DHAVE_BUILD_INFO -DUSE_LIBEVENT)
```

2. src/config/bitcoin-config.h

Przejście na nowy kompilator GCC 14 oraz bibliotekę systemową glibc w systemie Linux spowodowało konflikty i redefinicje wbudowanych funkcji (takich jak bswap czy strnlen). Utworzyliśmy scentralizowany plik konfiguracyjny, który dezaktywuje te konflikty, wymuszając flagi HAVE_DECL_*.
Zastosowany kod C++:


```bash
C++
#ifndef BITCOIN_CONFIG_H
#define BITCOIN_CONFIG_H

#define PACKAGE_NAME "Litecoin Core"
#define PACKAGE_VERSION "0.21.5.6"

/* Blokada redefinicji funkcji systemowych glibc */
#define HAVE_DECL_STRNLEN 1
#define HAVE_DECL_HTOBE16 1
#define HAVE_DECL_HTOLE16 1
#define HAVE_DECL_BE16TOH 1
#define HAVE_DECL_LE16TOH 1
#define HAVE_GMTIME_R 1

#endif // BITCOIN_CONFIG_H
```

Komenda do zastosowania:


```bash
Bash
mkdir -p src/config
nano src/config/bitcoin-config.h
# Wklej powyższy kod i zapisz plik
```

3. src/support/allocators/zeroafterfree.h oraz secure.h

Aktualizacja kompilatora do standardu C++20 sprawiła, że stare aliasy typów używane do bezpiecznego zarządzania pamięcią (np. base::pointer) przestały być obsługiwane. Kod musiał zostać napisany na nowo poprzez bezpośrednie zdefiniowanie wskaźników typu T*.
Zastosowany kod C++:


```bash
C++
template <typename T>
struct zero_after_free_allocator : public std::allocator<T> {
    typedef std::allocator<T> base;
    typedef typename base::size_type size_type;
    typedef typename base::difference_type difference_type;
    
    // Zgodność z C++20: Wprost zdefiniowane typy
    typedef T* pointer;
    typedef const T* const_pointer;
    typedef T& reference;
    typedef const T& const_reference;
    typedef typename base::value_type value_type;
    
    void deallocate(T* p, size_t n) {
        if (p != nullptr) memory_cleanse(p, sizeof(T) * n);
        std::allocator<T>::deallocate(p, n);
    }
};
```

4. src/libmw/include/mw/util/StringUtil.h

Nowsza wersja biblioteki formatującej tekst (fmt v10) wymaga, aby zmienne tekstowe w locie były oznaczane specjalną klasą fmt::runtime. Zabezpiecza to kod przed błędami bezpieczeństwa typu consteval.
Zastosowany kod C++:


```
C++
template<typename ... Args>
static std::string Format(const char* format, const Args&... args) {
    return fmt::format(fmt::runtime(format), ConvertArg(args)...);
}
```

Komenda automatycznie naprawiająca plik:



```bash
sed -i 's/return fmt::format(format, ConvertArg(args)...);/return fmt::format(fmt::runtime(format), ConvertArg(args)...);/' src/libmw/include/mw/util/StringUtil.h
```

5. src/leveldb/port/port_config.h

Wbudowany silnik bazy danych LevelDB wymagał oddzielnego pliku ucinającego archaiczne funkcjonalności platformy POSIX. Stworzyliśmy nowy nagłówek, który zdejmuje błędy kolejności bajtów (Endianness).
Zastosowany kod C++:


```bash
C++
#ifndef STORAGE_LEVELDB_PORT_PORT_CONFIG_H_
#define STORAGE_LEVELDB_PORT_PORT_CONFIG_H_
#define LEVELDB_IS_BIG_ENDIAN 0
#define HAVE_FDATASYNC 1
#define HAVE_O_CLOEXEC 1
#endif
```

6. src/leveldb/util/env_posix.cc

W ramach dopasowywania bazy LevelDB do standardów nowej generacji, należało usunąć wycofane prefiksy przy deklaracji zarządzania pamięcią. Stare std::memory_order::memory_order_relaxed zostało zaktualizowane.
Komenda automatycznie naprawiająca plik:



```bash
sed -i 's/std::memory_order::memory_order_relaxed/std::memory_order_relaxed/g' src/leveldb/util/env_posix.cc
```

Kompilacja finałowa
Gdy wszystkie pliki zostały zmodyfikowane i dostosowane do współczesnego środowiska (C++20), program jest gotowy do zbudowania za pomocą poleceń systemowych.
Aby poprawnie zainicjować budowę i utworzyć klienta RPC (litecoin-cli), wykonaj poniższe kroki w terminalu:



```bash
mkdir -p build_cmake
cd build_cmake
cmake ..
cmake --build . --target litecoin-cli -j$(nproc)
```


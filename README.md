# Catalyst
C++ factories with fully polymorphic object creation, custom RAII pointers, and perfect forwarding.

Hello, here's the culmination of a few days of learning and hard work on the concept of smart pointers and polymorphic factories. During this little adventure, I learned many things about a subject that I'm increasingly passionate about: metaprogramming with templates. The code you see is a combination of my Lumen project and the knowledge I've gained regarding expansion packs and variadic templates. If only I had enough words to explain the entire process behind such a project, it's not the result that counts, it's the journey, and I can't express how much fun I had during this work. Among these learnings, I can list some such as smart pointers and how they work, semantic movement with r and l values, variadic templates with the expansion pack, concepts about scope and object lifetime in C++, and incidentally, raw pointers, which I had already explored a year ago during my data structures courses at CEGEP. If any of you have comments, fixes, or simply want to discuss this fascinating topic with me, I'm waiting! I'd love to learn and meet people who share my passions. So, if you stumble upon this repo and are intrigued or a beginner, I'd be happy to answer your questions or explain how it all works. It's a win-win situation: you learn something, and I get the chance to spread the word about meta-programming :)

Finally, I also wanted to add that my code is far from perfect and is probably very unsafe and dangerous to use. In fact, I don't recommend using it at all outside of experimental projects or as an introduction to smart pointers, for the purpose of simplifying the system behind this wonderful feature of the standard library. The important thing here, I think, is that I've learned the fundamental things you need to know. Keep in mind that I only learned how smart pointers work last week and haven't officially used them in my projects yet, but that will probably change soon :) Well, if you're still reading this message, I wish you a very nice rest of the day!

```cpp
// Copyright (c) November 2025 Félix-Olivier Dumas. All rights reserved.
// Licensed under the terms described in the LICENSE file

#include <iostream>
#include <utility>

//literally just recycling my smart-ish smart_ptr library, would definitely
//not use that in production. I'm just doing it for fun :)
namespace nex {
    #ifdef NEX_DEBUG
        #ifdef NEX_VERBOSE_LOG
            #define NEX_LOG(msg) std::cout << "[Verbose]" << msg << std::endl
        #else
            #define NEX_LOG(msg) std::cout << msg << std::endl
        #endif
    #else
        #define NEX_LOG(msg)
    #endif

    template<typename T>
    struct is_lvalue_reference { static constexpr bool value = false; };

    template<typename T>
    struct is_lvalue_reference<T&> { static constexpr bool value = true; };

    template<typename T>
    struct is_rvalue_reference { static constexpr bool value = false; };

    template<typename T>
    struct is_rvalue_reference<T&&> { static constexpr bool value = true; };

    template<typename T>
    static inline T&& move(T& lvalue) { return static_cast<T&&>(lvalue); }

    template<typename T>
    static inline T&& forward(T& t) noexcept {
        return static_cast<T&&>(t);
    }

    template<typename T>
    class shared_ptr {
        unsigned int* _refCount = nullptr;

    private:
        T* _raw;

    public:
        shared_ptr(void) {
            NEX_LOG("[shared_ptr] Creating a new shared pointer");
            _raw = nullptr;
            _refCount = new unsigned int(1);
            std::cout << *_refCount << std::endl;
        }

        shared_ptr(T&& value)
            : _raw(new T(nex::move(value))),
            _refCount(new unsigned int(1)) {
        }

        template<typename U = T, typename... Args>
        explicit shared_ptr(Args&&... args)
            : _raw(new U(nex::forward<Args>(args)...)),
            _refCount(new unsigned int(1)) {
            std::cout << *_refCount << std::endl;
        }

        ~shared_ptr(void) {
            NEX_LOG("[~shared_ptr] Destroying the shared pointer");

            if (--(*_refCount) == 0) {
                NEX_LOG("[~shared_ptr] Refcount at 0, Destroying the shared value");
                delete _refCount;
                delete _raw;
            }
            _raw = nullptr;
        }

        shared_ptr<T>& operator=(const shared_ptr<T>& other) {
            std::cout << "Assignment operator called." << std::endl;

            if (this != &other) {
                if (_refCount && --(*_refCount) == 0) {
                    delete _raw;
                    delete _refCount;
                }

                _raw = other._raw;
                _refCount = other._refCount;

                if (_refCount) {
                    ++(*_refCount);
                }
            }
            return *this;
        }

        #ifndef __INTELLISENCE__
            T* operator->() { return _raw; }
            const T* operator->() const { return _raw; }
            T& operator*() { return *_raw; }
            T* get() const { return _raw; }
        #else
            T* get() const { return nullptr; }
        #endif // I know...

        shared_ptr(const shared_ptr& other) noexcept {
            NEX_LOG("[shared_ptr] Copying shared_ptr values to a new shared_ptr");

            _refCount = other._refCount;
            _raw = other._raw;

            if (_refCount)
                ++(*_refCount);
        }
    };


    template<typename T>
    class scoped_ptr {
    private:
        T* _raw;

    public:
        scoped_ptr(void) {
            _raw = nullptr;
            NEX_LOG("[scoped_ptr] Creating a new scope pointer");
        }
        scoped_ptr(T&& value) : _raw(new T(nex::move(value))) {}

        scoped_ptr(scoped_ptr&& other) noexcept {
            NEX_LOG("[scoped_ptr] Transferring scoped_ptr property");
            _raw = other._raw;
            other._raw = nullptr;
        }

        template<typename... Args>
        scoped_ptr(Args&&... args) : _raw(new T(nex::forward<Args>(args)...)) {}

        ~scoped_ptr(void) {
            NEX_LOG("[~scoped_ptr] Destroying the smart pointer");
            delete _raw; _raw = nullptr;
        }

        scoped_ptr<T>& operator=(scoped_ptr<T>&& other) {
            _raw = other._raw;
            other._raw = nullptr;

            return *this;
        }

        #ifndef __INTELLISENCE__
            T* operator->() { return _raw; }
            const T* operator->() const { return _raw; }
            T& operator*() { return *_raw; }
            T* get() const { return _raw; }
        #else
            T* get() const { return nullptr; }
        #endif
    };

    template<typename T, typename... Ts>
    inline static scoped_ptr<T> make_scoped_ptr(Ts&&... args) {
        std::cout << "[make_scoped_ptr] Making a new scoped pointer of " << typeid(T).name() << std::endl;
        return scoped_ptr<T>(nex::forward<Ts>(args)...);
    }

    template<typename T, typename... Ts>
    inline static shared_ptr<T> make_shared_ptr(Ts&&... args) {
        std::cout << "[make_shared_ptr] Making a new shared pointer of " << typeid(T).name() << std::endl;
        return shared_ptr<T>(nex::forward<Ts>(args)...);
    }
}

namespace Lib {
    namespace Dev {
        template<typename T>
        static inline T&& move(T& lvalue) { return static_cast<T&&>(lvalue); }

        template<typename T>
        static inline T&& forward(T& t) noexcept { return static_cast<T&&>(t); }

        class Test_Object {
        private:
            struct Defaults {
                static constexpr uint32_t A = 1;
                static constexpr double B = 10.0;
            };

        public:
            explicit Test_Object(uint32_t a = Defaults::A, double b = Defaults::B)
                : a_(a), b_(b) {
            }

            inline void methodA() const { printf("Value A: %i\n", a_); }
            inline void methodB() const { printf("Value B: %d\n", b_); }

        private:
            uint32_t a_;
            double b_;
        };


        class Factory {
        public:
            template<typename T, typename... Ts>
            constexpr T create(Ts&&... args) const noexcept {
                return T(std::forward<Ts&&>(args)...);
            }

            static Factory Initialize() noexcept {
                return Factory();
            }

        private:
            Factory() { }
        };
    }

    inline namespace Release {
        using Object = Lib::Dev::Test_Object;

        template<typename T, typename... Ts>
        static constexpr T Create(Ts&&... args) noexcept {
            return T(nex::forward<Ts&&>(args)...);
        }

        template<typename T, typename... Ts>
        static constexpr T* create_raw_ptr(Ts&&... args) noexcept {
            return new T(nex::forward<Ts&&>(args)...);
        }

        // This is intentional standard code, no need to panic, it doesn't have
        // the same responsibility as the internal construction of scoped pointers.
        template<typename T, typename... Ts>
        static constexpr nex::scoped_ptr<T> create_scoped_ptr(Ts&&... args) noexcept {
            return nex::make_scoped_ptr<T>(nex::forward<Ts&&>(args)...);
        }
    }
}

using namespace Lib; int main() {
    { // DEV: A common and well-implemented factory with polymorphic instantiation
        auto factory = Dev::Factory::Initialize();
        auto instance = factory.create<Dev::Test_Object>(10, 50.0);

        instance.methodA();
        instance.methodB();
    } // DEV: object lifecycle ends here
    

    { // RELEASE: clean and type-safe object creation with polymorphic support
        {
            // Stack allocation
            auto instance = Create<Object>(100, 10.0);
            instance.methodA();
            instance.methodB();
        }
        {
            // Raw pointer
            auto instancePtr = create_raw_ptr<Object>(200, 20.0);
            instancePtr->methodA();
            instancePtr->methodB();
            delete instancePtr;
        }
        {
            // Unique-ish pointer
            auto instanceUnique = create_scoped_ptr<Object>(300, 30.0);
            instanceUnique->methodA();
            instanceUnique->methodB();
        }
    } // RELEASE: object lifecycle ends here
}
```

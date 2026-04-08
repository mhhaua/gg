# gg
// =============================================
// libfpslock.so - FPS Stabilizer for Free Fire
// Phù hợp Android 15 (arm64) - External
// Compile với NDK cho arm64-v8a
// =============================================

#include <thread>
#include <chrono>
#include <cmath>
#include "KittyMemoryEx.hpp"

namespace Offsets {
    // === OFFSET CẦN DUMP LẠI MỚI NHẤT (dùng Il2CppDumper) ===
    uintptr_t UNITY_APPLICATION_TARGETFRAMERATE = 0x0;     // Thay offset thật
    uintptr_t UNITY_TIME_DELTA_TIME             = 0x0;     // Thay offset thật
    uintptr_t UNITY_QUALITY_SETTINGS            = 0x0;     // Thay offset thật
    uintptr_t VSYNC_COUNT                       = 0x0;     // Thay offset thật
}

struct FPSConfig {
    bool enabled = true;
    int targetFPS = 60;           // 60 / 90 / 120 (tùy máy bạn hỗ trợ)
    bool forceHighRefresh = true; // Buộc refresh rate cao
    float minDeltaTime = 0.008f;  // \~120 FPS max
    bool antiStutter = true;      // Giảm giật frame
};

FPSConfig cfg;

void FPSStabilizerThread() {
    KittyMemoryEx mem;
    
    while (!mem.init("com.dts.freefireth")) {
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }

    uintptr_t il2cppBase = mem.getModuleBase("libil2cpp.so");
    if (!il2cppBase) return;

    std::cout << "[+] FPS Stabilizer Attached - Target: " << cfg.targetFPS << " FPS\n";

    while (cfg.enabled) {
        // === 1. Force Target Frame Rate ===
        if (Offsets::UNITY_APPLICATION_TARGETFRAMERATE) {
            int currentFPS = mem.read<int>(il2cppBase + Offsets::UNITY_APPLICATION_TARGETFRAMERATE);
            if (currentFPS != cfg.targetFPS) {
                mem.write<int>(il2cppBase + Offsets::UNITY_APPLICATION_TARGETFRAMERATE, cfg.targetFPS);
            }
        }

        // === 2. Force VSync Off (giảm giật) ===
        if (Offsets::VSYNC_COUNT) {
            mem.write<int>(il2cppBase + Offsets::VSYNC_COUNT, 0);
        }

        // === 3. Stabilize Delta Time (chống drop frame) ===
        if (Offsets::UNITY_TIME_DELTA_TIME) {
            float delta = mem.read<float>(il2cppBase + Offsets::UNITY_TIME_DELTA_TIME);
            if (delta > cfg.minDeltaTime) {
                mem.write<float>(il2cppBase + Offsets::UNITY_TIME_DELTA_TIME, cfg.minDeltaTime);
            }
        }

        // === 4. Quality Settings tối ưu (Android 15) ===
        if (Offsets::UNITY_QUALITY_SETTINGS) {
            // Set maximum quality + anisotropic filtering
            mem.write<int>(il2cppBase + Offsets::UNITY_QUALITY_SETTINGS + 0x10, 5); // Graphics tier
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(8)); // Loop \~120 lần/giây
    }
}

// Hàm chính được gọi khi load .so
extern "C" void _init() {
    std::thread(FPSStabilizerThread).detach();
}

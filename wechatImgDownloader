from pywinauto import Application
import time
import os
import hashlib
import threading
from pywinauto.keyboard import send_keys
from collections import defaultdict

# 配置参数
SAVE_FOLDER = r"C:\Users\e3724\Desktop\a"  # 目标保存路径
MAX_REPEAT = 2  # MD5重复阈值（达到此次数则停止）
WAIT_DELAY = 0.01  # 操作延迟（秒）
MONITOR_INTERVAL = 1  # 监控线程扫描间隔（秒）

# 全局变量：文件名→MD5映射（线程安全）
file_md5_map = {}
# 全局变量：MD5出现次数统计
md5_count = defaultdict(int)
# 线程锁
map_lock = threading.Lock()

def calculate_md5(file_path):
    """计算文件MD5值"""
    if not os.path.exists(file_path):
        return None
    try:
        hash_md5 = hashlib.md5()
        with open(file_path, "rb") as f:
            for chunk in iter(lambda: f.read(4096), b""):
                hash_md5.update(chunk)
        return hash_md5.hexdigest()
    except Exception as e:
        print(f"计算MD5失败：{str(e)}")
        return None

def folder_monitor():
    """后台线程：监控文件夹并更新MD5映射"""
    global file_md5_map, md5_count
    while True:
        if os.path.exists(SAVE_FOLDER):
            for filename in os.listdir(SAVE_FOLDER):
                if filename.endswith(".jpg") and filename.split(".")[0].isdigit():
                    file_path = os.path.join(SAVE_FOLDER, filename)
                    with map_lock:
                        current_md5 = calculate_md5(file_path)
                        if current_md5:
                            # 处理文件更新（如覆盖保存）
                            if filename in file_md5_map:
                                old_md5 = file_md5_map[filename]
                                md5_count[old_md5] -= 1
                                if md5_count[old_md5] <= 0:
                                    del md5_count[old_md5]
                            # 更新映射和计数
                            file_md5_map[filename] = current_md5
                            md5_count[current_md5] += 1
        time.sleep(MONITOR_INTERVAL)

def wait_for_file(filename, timeout=10):
    """等待文件被监控线程捕获"""
    start_time = time.time()
    while time.time() - start_time < timeout:
        with map_lock:
            if filename in file_md5_map:
                return True
        time.sleep(0.5)
    return False

def final_dedup():
    """流程结束后统一删除重复文件（只保留第一个出现的）"""
    with map_lock:
        # 按文件名排序（确保按顺序保留第一个）
        sorted_files = sorted(file_md5_map.keys(), key=lambda x: int(x.split(".")[0]))
        md5_first_occurrence = {}  # 记录MD5首次出现的文件名
        duplicate_files = []       # 记录重复文件

        for filename in sorted_files:
            md5 = file_md5_map[filename]
            if md5 not in md5_first_occurrence:
                md5_first_occurrence[md5] = filename
            else:
                duplicate_files.append(filename)

        # 删除重复文件
        for filename in duplicate_files:
            file_path = os.path.join(SAVE_FOLDER, filename)
            if os.path.exists(file_path):
                try:
                    os.remove(file_path)
                    print(f"最终去重：删除重复文件 {filename}")
                except Exception as e:
                    print(f"删除重复文件失败 {filename}：{str(e)}")

        return len(sorted_files) - len(duplicate_files), len(duplicate_files)

def auto_download_wechat_images(window_title, next_btn_text="下一张", save_btn_text="另存为"):
    # 记录开始时间
    start_time = time.time()
    try:
        # 启动监控线程
        monitor_thread = threading.Thread(target=folder_monitor, daemon=True)
        monitor_thread.start()
        print("已启动文件夹监控线程...")
        time.sleep(2)

        # 连接窗口
        app = Application(backend="uia").connect(title_re=f".*{window_title}.*", timeout=10)
        main_window = app.window(title_re=f".*{window_title}.*")
        main_window.set_focus()
        print(f"已连接到窗口：{main_window.window_text()}")

        click_count = 1
        stop_flag = False

        while not stop_flag:
            # 查找下一张按钮
            next_btn = None
            for btn in main_window.descendants(control_type="Button"):
                btn_text = btn.window_text().strip()
                if next_btn_text in btn_text or "→" in btn_text:
                    next_btn = btn
                    break

            if not next_btn or not next_btn.is_enabled():
                print("未找到可用的下一张按钮，停止流程")
                break

            # 查找另存为按钮
            save_btn = None
            for btn in main_window.descendants(control_type="Button"):
                btn_text = btn.window_text().strip()
                if save_btn_text in btn_text or "保存" in btn_text:
                    save_btn = btn
                    break

            if not save_btn or not save_btn.is_enabled():
                print("未找到可用的另存为按钮，跳过")
                next_btn.click_input()
                click_count += 1
                time.sleep(WAIT_DELAY)
                continue

            # 保存当前图片
            current_filename = f"{click_count}.jpg"
            print(f"\n===== 处理第{click_count}张图片（{current_filename}）=====")
            save_btn.click_input()
            time.sleep(WAIT_DELAY )  # 等待对话框

            # 输入文件名（重试机制）
            save_success = False
            for _ in range(3):
                try:
                    send_keys(current_filename.split(".")[0])  # 输入数字
                    time.sleep(0.5)
                    send_keys("{ENTER}")
                    save_success = True
                    break
                except:
                    time.sleep(1)

            if not save_success:
                print("保存失败，跳过")
                next_btn.click_input()
                click_count += 1
                continue

            # 等待文件被监控线程捕获
            if not wait_for_file(current_filename):
                print(f"警告：{current_filename} 未被监控到，可能保存路径错误")
                next_btn.click_input()
                click_count += 1
                continue

            # 检查MD5重复情况
            with map_lock:
                current_md5 = file_md5_map[current_filename]
                current_md5_count = md5_count[current_md5]

            print(f"当前MD5：{current_md5}，累计出现次数：{current_md5_count}")

            # 首次重复时输出日志
            if current_md5_count == 2:
                print(f"⚠️ MD5首次重复（当前文件：{current_filename}）")
            # 达到重复阈值时停止流程
            elif current_md5_count >= MAX_REPEAT:
                print(f"⚠️ MD5重复达到阈值（{MAX_REPEAT}次），停止流程")
                stop_flag = True
                break

            # 点击下一张
            next_btn.click_input()
            click_count += 1
            time.sleep(WAIT_DELAY * 2)

        # 流程结束后统一去重
        print("\n开始最终去重...")
        total_valid, total_duplicate = final_dedup()
        
        # 计算总耗时
        end_time = time.time()
        total_time = end_time - start_time
        minutes = int(total_time // 60)
        seconds = total_time % 60

        print(f"\n===== 结果统计 =====")
        print(f"总下载文件数：{len(file_md5_map)}")
        print(f"去重后保留：{total_valid} 张")
        print(f"删除重复文件：{total_duplicate} 张")
        print(f"总耗时：{minutes}分{seconds:.2f}秒")

    except Exception as e:
        # 异常时也统计耗时
        end_time = time.time()
        total_time = end_time - start_time
        print(f"流程出错：{str(e)}，已运行{total_time:.2f}秒")

if __name__ == "__main__":
    input("请确认路径正确且手动测试过保存功能，按回车开始...")

    auto_download_wechat_images(
        window_title="图片查看",
        next_btn_text="下一张",
        save_btn_text="另存为"
    )

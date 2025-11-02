import pyautogui
import time
import os
import subprocess
import hashlib

# 配置参数
SAVE_BUTTON_IMG = "save_button.png"  # 保存按钮截图
WAIT_TIME = 0.3  # 操作间隔隔（秒）
OUTPUT_DIR = os.path.expanduser("/home/swt/桌面/a")  # 保存目录
FILE_COUNTER = 1  # 文件名计数器（从1开始）
RECENT_FILES = []  # 最近保存的文件路径


def get_file_md5(file_path):
    """计算文件MD5"""
    if not os.path.exists(file_path):
        return None
    md5_hash = hashlib.md5()
    with open(file_path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            md5_hash.update(chunk)
    return md5_hash.hexdigest()


def ensure_output_dir():
    """确保保存目录存在"""
    os.makedirs(OUTPUT_DIR, exist_ok=True)
    print(f"图片将保存到：{OUTPUT_DIR}")


def find_wechat_image_window():
    """通过wmctrl查找微信图片窗口"""
    try:
        result = subprocess.check_output(
            "wmctrl -l | grep -i '图片'",  # 替换为实际窗口标题关键词
            shell=True,
            text=True
        )
        return result.split()[0]  # 返回窗口ID
    except subprocess.CalledProcessError:
        print("未找到微信图片窗口，请先打开图片查看模式")
        return None


def activate_window(window_id):
    """激活窗口"""
    subprocess.run(f"wmctrl -i -a {window_id}", shell=True)
    time.sleep(WAIT_TIME)


def click_save_button():
    """点击保存按钮"""
    try:
        button_pos = pyautogui.locateOnScreen(
            SAVE_BUTTON_IMG,
            confidence=0.7,
            grayscale=True
        )
        if button_pos:
            pyautogui.click(pyautogui.center(button_pos))
            print("已点击保存按钮")
            return True
        print("未找到保存按钮")
        return False
    except Exception as e:
        print(f"按钮识别错误：{e}")
        return False


def handle_save_dialog():
    """处理保存对话框，直接输入纯数字文件名"""
    global FILE_COUNTER
    time.sleep(WAIT_TIME)  # 等待对话框弹出

    # 1. 全选默认文件名（确保覆盖）
    pyautogui.hotkey("ctrl", "a")
    #time.sleep(0.5)

    # 2. 直接输入当前计数器（纯数字，不带扩展名）
    pyautogui.typewrite(str(FILE_COUNTER))
    #time.sleep(0.5)

    # 3. 按Enter确认保存
    pyautogui.press("enter")
    print(f"已保存为：{FILE_COUNTER}")

    # 4. 记录文件路径并递增计数器
    # 注：实际保存的文件会自动带扩展名（微信默认添加）
    saved_path = os.path.join(OUTPUT_DIR, f"{FILE_COUNTER}")  # 后续系统会自动补全扩展名
    RECENT_FILES.append(saved_path)
    FILE_COUNTER += 1  # 下次保存+1
    time.sleep(WAIT_TIME)


def next_image():
    """切换到下一张图片"""
    pyautogui.press("right")
    print("切换到下一张图片")
    time.sleep(WAIT_TIME)


def check_md5_duplicate():
    """检查最近2张图片MD5是否一致"""
    if len(RECENT_FILES) >= 2:
        # 补全实际文件名（微信会自动添加扩展名，如.png）
        file1 = find_actual_file(RECENT_FILES[-2])
        file2 = find_actual_file(RECENT_FILES[-1])
        if not file1 or not file2:
            return False
        md5_1 = get_file_md5(file1)
        md5_2 = get_file_md5(file2)
        if md5_1 and md5_2 and md5_1 == md5_2:
            print(f"\n检测到连续2张图片MD5一致：{md5_1}")
            return True
    return False


def find_actual_file(base_name):
    """查找带扩展名的实际文件（如输入1，实际可能是1.png）"""
    for ext in [".png", ".jpg", ".jpeg"]:  # 常见图片扩展名
        candidate = f"{base_name}{ext}"
        if os.path.exists(candidate):
            return candidate
    return None  # 未找到对应文件


def batch_save_images(max_count=20):
    """批量保存图片，支持纯数字文件名和重复检测"""
    ensure_output_dir()
    window_id = find_wechat_image_window()
    if not window_id:
        return

    activate_window(window_id)

    for i in range(max_count):
        print(f"\n处理第{i+1}/{max_count}张图片")
        if click_save_button():
            handle_save_dialog()
            # 检查连续2张是否重复
            if check_md5_duplicate():
                print("程序终止：连续2张图片重复")
                return
        else:
            print("跳过当前图片")
        next_image()

    print("\n批量保存完成（已达最大数量）")


if __name__ == "__main__":
    input("请确保已打开微信图片查看窗口，按Enter开始...")
    batch_save_images(max_count=20)

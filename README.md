# Save-Code-AI
Code Predict AI:
import cv2
from ultralytics import YOLO
import os
import threading
from tkinter import Tk, Label, Entry, messagebox, PhotoImage
from tkinter import ttk
import serial
import time
import sys
from datetime import datetime

# Nhận camera từ app main
if len(sys.argv) > 1:
    camera_index = int(sys.argv[1])
else:
    camera_index = 0

# Biến trạng thái
ng_text = ""  # Thông báo lỗi
button_active = False  # Biến để kiểm soát cửa sổ nút bấm
error_text = ""
com_port = "COM4"
turn_off_led = "02"
turn_on_led = "01"

# DEBUG
debug_mode = False

# Chup anh loi
def task_capture_4k(error_type="Unknow"):
    global cap, camera_index, is_capturing
    is_capturing = True  # Khóa vòng lặp chính lại để không tranh chấp camera

    print(f"--- Đang chuyển sang 4K để chụp lỗi: {error_type} ---")

    try:
        # 1. Giải phóng camera 720p hiện tại ngay lập tức
        if cap.isOpened():
            cap.release()

        # 2. Mở camera ở chế độ 4K bằng một biến tạm (Local variable)
        temp_cap = cv2.VideoCapture(camera_index, cv2.CAP_DSHOW)
        temp_cap.set(cv2.CAP_PROP_FRAME_WIDTH, 3840)
        temp_cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 2160)

        # 3. Thay vì sleep(1), ta dùng grab() để "xả" frame cũ cực nhanh
        # i7 Gen 12 xử lý việc này chỉ mất khoảng 200-300ms
        for _ in range(3):
            temp_cap.grab()

        ret, frame = temp_cap.read()
        if ret:
            save_folder = "box_error_images"
            os.makedirs(save_folder, exist_ok=True)
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = os.path.join(save_folder, f"{error_type}_{timestamp}.jpg")
            cv2.imwrite(filename, frame)
            print("--- Đã lưu ảnh 4K thành công! ---")

        temp_cap.release()  # Đóng 4K

    except Exception as e:
        print(f"Lỗi khi chụp 4K: {e}")

    finally:
        # 4. Mở lại camera 720p và gán lại cho biến toàn cục 'cap'
        # Việc này giúp vòng lặp chính tự động chạy lại mượt mà
        cap = cv2.VideoCapture(camera_index, cv2.CAP_DSHOW)
        cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1024)
        cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 576)

        is_capturing = False  # Mở khóa cho vòng lặp chính
        print("--- Đã trả lại camera 720p cho AI ---")

# Xau thuc mat khau
def verify_password(event=None):  # Thêm event=None để hỗ trợ cả nút bấm và phím Enter
        global root, current_task
        entered_password = password_entry.get()  # Lấy mật khẩu nhập vào
        correct_password = "1"  # Mật khẩu đúng
        reset_task_password = "2"

        if entered_password == correct_password:
            stop_and_clear()  # Gọi hàm xử lý nếu mật khẩu đúng

        # Mat khau Reset task
        elif entered_password == reset_task_password:
            current_task = "Put Box In"
            stop_and_clear()
        else:
            messagebox.showerror("Sai mật khẩu", "Mật khẩu không chính xác. Vui lòng thử lại!")  # Hiển thị lỗi

def show_stop_button():
        global root, password_entry

        # Kiểm tra xem root đã tồn tại hay chưa
        # if root and root.winfo_exists():
        #     root.destroy()  # Đảm bảo root cũ được xoá hoàn toàn trước khi tạo mới

        # Khởi tạo root mới
        root = Tk()
        root.configure(background="red")  # Nền đỏ để tạo cảm giác cảnh báo
        root.after(100, check_and_show_button)
        root.title("Thông báo lỗi")

        # Đặt kích thước cửa sổ và căn giữa
        window_width, window_height = 800, 600
        screen_width = root.winfo_screenwidth()
        screen_height = root.winfo_screenheight()
        position_top = (screen_height // 2) - (window_height // 2)
        position_right = (screen_width // 2) - (window_width // 2)
        root.geometry(f"{window_width}x{window_height}+{position_right}+{position_top}")

        # Thêm biểu tượng cảnh báo
        try:
            warning_icon = PhotoImage(file="warning_icon.png")  # Đảm bảo file icon tồn tại
            Label(root, image=warning_icon, bg="red").place(relx=0.5, rely=0.2, anchor="center")
            root.warning_icon = warning_icon  # Giữ biểu tượng để không bị thu hồi
        except:
            Label(root, text="⚠️", font=("Helvetica", 50, "bold"), fg="white", bg="red").place(relx=0.5, rely=0.2, anchor="center")

        # Thêm nhãn thông báo
        Label(root, text=f"{error_text}. Nhập mật khẩu để xác thực:",
              font=("Arial", 16, "bold"), fg="white", bg="red").place(relx=0.5, rely=0.4, anchor="center")

        # Entry để nhập mật khẩu
        password_entry = Entry(root, show="*", font=("Arial", 14), bg="white", fg="black", justify="center")
        password_entry.place(relx=0.5, rely=0.5, anchor="center")
        # Đặt focus vào Entry sau khi khởi tạo toàn bộ GUI
        root.after(100, lambda: password_entry.focus_set())
        password_entry.bind("<Return>", verify_password)  # Tự động kết nối phím Enter với verify_password

        # Nút xác thực mật khẩu
        ttk.Style().configure("TButton", font=("Arial", 14), padding=10)
        ttk.Button(root, text="Xác thực", command=verify_password).place(relx=0.5, rely=0.65, anchor="center")

        root.protocol("WM_DELETE_WINDOW", lambda: None)
        root.mainloop()

def stop_and_clear():
        global root, stop_music, button_active, put_box_in_time_elapsed,check_enough_parts_time_elapsed,missing_part_time_elapsed,put_box_out_time_elapsed, last_time

        put_box_in_time_elapsed = 0.0
        check_enough_parts_time_elapsed = 0.0
        missing_part_time_elapsed = 0.0
        put_box_out_time_elapsed = 0.0

        last_time = time.time()

        try:
            # Tắt LED
            with serial.Serial(com_port, 9600, timeout=1) as ser:  # Thay 'COM3' bằng cổng bạn sử dụng
                ser.write(bytes.fromhex(turn_off_led))
                ser.write(bytes.fromhex(turn_off_led))
                ser.write(bytes.fromhex(turn_off_led))
                ser.write(bytes.fromhex(turn_off_led))
                ser.write(bytes.fromhex(turn_off_led))

                if ser.write(bytes.fromhex(turn_off_led)):  # Gửi lệnh HEX để tắt LED
                    print("Lệnh HEX gửi thành công!")
                else:
                    print("Không gửi được lệnh HEX")

        except serial.SerialException as e:
            print(f"Lỗi kết nối Serial: {e}")
        except Exception as e:
            print(f"Lỗi không xác định khi tắt LED: {e}")

        # Cập nhật trạng thái
        button_active = False

        root.destroy()
        # try:
        #     if root and root.winfo_exists():  # Kiểm tra xem root có tồn tại không
        #         root.destroy()  # Đóng cửa sổ
        #         print("Cửa sổ đã được đóng.")
        # except Exception as e:
        #     print(f"Lỗi khi đóng cửa sổ: {e}")
        #
        # print("Xử lý xong lỗi và dừng nhạc!")

def check_and_show_button():
        global button_active
        if not button_active:  # Kiểm tra trạng thái nút bấm
            button_active = True
            show_stop_button()  # Hiển thị giao diện
        root.after(100, check_and_show_button)  # Kiểm tra lặp lại

# Ham xu ly loi
def handle_error(error_message):
        global error_text
        error_text = error_message

        # Bật LED
        try:
            with serial.Serial(com_port, 9600, timeout=1) as ser:  # Thay 'COM3' bằng cổng bạn sử dụng
                ser.write(bytes.fromhex(turn_on_led))
                ser.write(bytes.fromhex(turn_on_led))
                ser.write(bytes.fromhex(turn_on_led))
                ser.write(bytes.fromhex(turn_on_led))
                ser.write(bytes.fromhex(turn_on_led))

            # Gửi lệnh HEX để bật LED
                if not ser.write(bytes.fromhex(turn_on_led)):
                    print("Không gửi được lệnh HEX")
        except serial.SerialException as e:
            print(f"Lỗi kết nối Serial: {e}")
        except Exception as e:
            print(f"Lỗi không xác định khi bật LED: {e}")

        t = threading.Thread(target=task_capture_4k, args=(error_message,), daemon=True)
        t.start()

        check_and_show_button()

# Thiet lap model
IMG_SIZE = [576, 1024]
model_path = os.path.join(os.getcwd(),"_internal" ,"model_rm3_1674_openvino_model")
model = YOLO(model_path, task="obb")
# cap = cv2.VideoCapture(camera_index)
cap = cv2.VideoCapture(camera_index, cv2.CAP_DSHOW)

cap.set(cv2.CAP_PROP_FRAME_WIDTH, 1024)  # Độ rộng khung hình
cap.set(cv2.CAP_PROP_FRAME_HEIGHT, 576)  # Độ cao khung hình

# Giao dien
font = cv2.FONT_HERSHEY_SIMPLEX
font_scale = 0.7
font_thickness = 2

# Xác định nhiệm vụ
expected_tasks = [
    "Put Box In",
    "Check Enough Parts",
    "Put Box Out"
]

previous_task = None
current_task = "Put Box In"

required_parts = 4

# Biến thời gian. Star Platinum : The World
# 1. Khởi tạo biến lưu trữ thời điểm của vòng lặp trước
# Ta lấy thời điểm hiện tại làm mốc bắt đầu
last_time = time.time()

# OK
put_box_in_time_elapsed = 0.0
check_enough_parts_time_elapsed = 0.0
put_box_out_time_elapsed = 0.0

put_box_in_required_time = 1
check_enough_parts_required_time = 1
put_box_out_required_time = 1

# NG
ng_put_box_out_time_elapsed = 0.0
ng_missing_part_time_elapsed = 0.0

ng_put_box_out_required_time = 3
ng_missing_part_required_time = 3

# Full man hinh
cv2.namedWindow("AI Check RM3-1674", cv2.WINDOW_NORMAL)
cv2.resizeWindow("AI Check RM3-1674", 1024, 576)

is_capturing = False

while cap.isOpened():
    if is_capturing:
        cv2.waitKey(100)  # Nghỉ 0.1s
        continue

    ret, frame = cap.read()

    # # Lấy độ phân giải hiện tại của màn hình
    # screen_width, screen_height = pyautogui.size()
    # frame_height, frame_width = frame.shape[:2]
    # scale_width = screen_width / frame_width
    # scale_height = screen_height / frame_height
    # scale = min(scale_width, scale_height)
    #
    # resized_width = int(frame_width * scale)
    # resized_height = int(frame_height * scale)
    # resized_frame = cv2.resize(frame, (resized_width, resized_height), interpolation=cv2.INTER_AREA)
    #
    # # Tạo viền đen để vừa màn hình
    # border_top = (screen_height - resized_height) // 2
    # border_bottom = screen_height - resized_height - border_top
    # border_left = (screen_width - resized_width) // 2
    # border_right = screen_width - resized_width - border_left
    #
    # padded_frame = cv2.copyMakeBorder(resized_frame, border_top, border_bottom, border_left, border_right,
    #                                   cv2.BORDER_CONSTANT, value=[0, 0, 0])

    if not ret or frame is None:
        print("Frame lỗi - bỏ qua")
        continue

    current_time = time.time()
    delta_time = current_time - last_time
    last_time = current_time

    results = model.predict(frame, stream=False, conf=0.7, imgsz=IMG_SIZE, vid_stride=4)

    detected_objects = []

    for box in results[0].obb:
        x1, y1, x2, y2 = box.xyxy[0]
        class_id = int(box.cls[0])
        object_name = results[0].names[class_id]
        detected_objects.append(object_name)

    # Đếm số lượng các slot OK theo từng hướng
    part_count = detected_objects.count("part")

    frame_with_box = results[0].plot()

    # Buoc 1
    if current_task == "Put Box In":
        if "opened_box" in detected_objects:
            put_box_out_time_elapsed += delta_time
            if put_box_out_time_elapsed >= required_parts:
                put_box_in_time_elapsed = 0
                current_task = "Check Enough Parts"
        else:
            put_box_out_time_elapsed = 0
    #endregion

    # Buoc 2
    elif current_task == "Check Enough Parts":
        if part_count == required_parts:
            check_enough_parts_time_elapsed += delta_time
            if check_enough_parts_time_elapsed == required_parts:
                check_enough_parts_time_elapsed = 0
                current_task = "Put All Part Out"
        else:
            check_enough_parts_time_elapsed = 0

        if len(detected_objects) == 0:
            ng_put_box_out_time_elapsed += delta_time
            if ng_put_box_out_time_elapsed >= ng_put_box_out_required_time:
                ng_put_box_out_time_elapsed = 0
                handle_error("Bỏ hộp ra ngoài quá sớm")
        else:
            ng_put_box_out_time_elapsed = 0
    #endregion

    # Buoc 3
    elif current_task == "Put All Part Out":
        if len(detected_objects) == 0:
            ng_put_box_out_time_elapsed += delta_time
            if ng_put_box_out_time_elapsed >= ng_put_box_out_required_time:
                ng_put_box_out_time_elapsed = 0
                current_task = "Put Box In"
        else:
            ng_put_box_out_time_elapsed = 0

        if part_count < required_parts:
            ng_missing_part_time_elapsed += delta_time
            if ng_missing_part_time_elapsed >= ng_missing_part_required_time:
                ng_missing_part_time_elapsed = 0
                current_task = "Check Enough Parts"
        else:
            ng_missing_part_time_elapsed = 0
    #endregion

    # Hien thi giao dien
    overlay = frame.copy()
    alpha = 0.5  # Độ trong suốt (0.0 hoàn toàn trong suốt, 1.0 là màu gốc)
    cv2.rectangle(overlay, (0, 0), (400, 320), (50, 50, 50), -1)  # Vẽ hình chữ nhật
    cv2.addWeighted(overlay, alpha, frame_with_box, 1 - alpha, 0, frame_with_box)  # Thêm hiệu ứng trong suốt
    cv2.putText(frame_with_box, f"Task : {current_task}", (30, 25), font, font_scale, (50, 255, 100), font_thickness)
    cv2.putText(frame_with_box, f"Parts: : {part_count}", (30, 55), font, font_scale, (50, 255, 100), font_thickness)
    cv2.putText(frame_with_box, f"Made By Nghia IT", (30, 85), font, font_scale, (0, 0, 255),
                font_thickness)
    cv2.putText(frame_with_box, f"DEBUG MODE: {debug_mode}", (30, 125), font, font_scale, (0, 0, 255),
                font_thickness)
    cv2.imshow("AI Check RM3-1674", frame_with_box)

    # Thoat chuong trinh
    key = cv2.waitKey(1) & 0xFF

    if key == ord('q'):
        break

    elif key == ord('d'):
        debug_mode = not debug_mode
        print("Debug mode:", debug_mode)

    # elif key == ord("e"):
    #     current_task = "Put All Part Out"

    elif key == ord("c"):
        task_capture_4k()

cap.release()
cv2.destroyAllWindows()

Code Train AI:
from ultralytics import YOLO

if __name__ == "__main__":
    model = YOLO(r"D:\Train AI\yolo11n-obb.pt")
    model.train(
        data= r"D:\Nghia Dataset\Packing Nagahama\Packing 4\RM3-1642-Dataset\data.yaml",
        epochs=500,
        imgsz=1024,
        batch=8,
        device=0,
        cos_lr=True,
        pretrained=True,
        project="runs/obb",
        name="rm3_1642_packing_4",
        optimizer="AdamW",
        lr0=0.001,
        patience=100,
        verbose=True,
        close_mosaic=100,

        # Augmentation cho hộp nghiêng
        augment=True,
        degrees=20.0, # Xoay ngau nhien
        mosaic=0.8, # Bat toi da de hoc vat the nho va khit nhau
        mixup=0.1, # Tron anh de chong hoc vet
        scale=0.5, # Phong to/ thu nho de hoc ca khi camera xa / gan
        flipud=0.5, # Thêm lật ảnh dọc vì linh kện có thể nhìn từ 2 phía
        fliplr=0.5 # Lật ảnh ngang
    )

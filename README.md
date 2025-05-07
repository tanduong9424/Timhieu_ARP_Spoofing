# Tìm hiểu phương pháp tấn công ARP spoofing
Video demo kết quả: https://drive.google.com/file/d/1a7-xC8VH5uqeKXG2GPDaJvr6yBN0Hkkb/view
## Yêu cầu tối thiểu về nội dung lý thuyết:
- Tổng quan về ARP spoofing (khái niệm, phân loại, cách tiến hành, ...).
- Mối liên hệ giữa phương pháp tấn công này với một số phương pháp tấn công khác.
- Cách phòng chống tấn công.
## Yêu cầu tối thiểu về nội dung demo:
- Demo 1 phương pháp tấn công có sử dụng ARP spoofing.
- Demo giải pháp phòng chống.

## Thành viên
- Hồ Hữu Đại       3122410066
- Huỳnh Tấn Dương  3122410061

MÔ HÌNH MẠNG
![](topology.png)

## I. CẤU HÌNH DHCP Ở ROUTER
- ```config terminal```
- ```ip dhcp excluded-address 192.168.1.1 192.168.1.5``` (Không cấp dãy ip 192.168.1.1->192.168.1.5 cho client)
- ```ip dhcp pool DHCP_ARP``` (đặt tên cho pool DHCPDHCP)
- ```network 192.168.1.0 255.255.255.0``` (dãy mạng được dùng để cấp phát)
- ```default-router 192.168.1.1``` (chỉ định default gatewaygateway)
- ```dns-server 8.8.8.8 1.1.1.1``` (chỉ định DNS serverserver)
- ```end```
- ```show run | s dhcp``` (kiểm tra DHCP đã hoạt động chưa)
## II. CẤU HÌNH Ở SWITCH
- ```config terminal```
- ```int vlan 1```
- ```ip add 192.168.1.2 255.255.255.0```
- ```no shut```
- ```exit```
- ```no ip dhcp snooping information option```
- ```int g0/0``` (cấu hình cổng từ router đến switch là trust, không cần kiểm tra gói tin đi qua đường này )
- ```ip dhcp snooping trust```

## IIII. Nguyên lý phòng chống
- DHCP Snooping xây dựng một cơ sở dữ liệu ánh xạ (binding table) giữa địa chỉ IP, MAC, cổng switch và VLAN, dựa trên các gói tin DHCP hợp lệ khi có yêu cầu cấp phát địa chỉ IP từ client đến DHCP server.
- Dynamic ARP Inspection (DAI) là một cơ chế bảo mật dùng để ngăn chặn ARP Spoofing, nó kiểm tra tính hợp lệ của các gói ARP bằng cách so sánh thông tin trong gói ARP với cơ sở dữ liệu binding của DHCP Snooping.

- So sánh IP-MAC trong gói ARP với thông tin trong DHCP Snooping binding table:
	- Nếu đúng: cho phép gói ARP đi qua.
	- Nếu sai: chặn gói ARP lại và có thể log hoặc báo động.
- Ngăn chặn kẻ tấn công giả mạo địa chỉ MAC/IP để đánh lừa client hoặc gateway.

- Trong tấn công ARP Spoofing, kẻ tấn công sẽ gửi các gói ARP Reply giả mạo với nội dung:
	- **(MAC của kẻ tấn công – IP của nạn nhân) gửi đến Gateway**
	- **(MAC của kẻ tấn công – IP của Gateway) gửi đến nạn nhân**

→ Mục đích là đánh lừa cả hai bên, khiến lưu lượng đi qua kẻ tấn công.

- Tuy nhiên, nếu switch được bật DHCP Snooping + DAI, thì các gói ARP này sẽ bị kiểm tra. Switch sẽ so sánh thông tin trong gói ARP với bảng DHCP Snooping Binding Table (chứa cặp MAC-IP hợp lệ).
- Do trong bảng chỉ có cặp hợp lệ là (MAC của kẻ tấn công – IP của kẻ tấn công), nên nếu gói ARP khai là (MAC của kẻ tấn công – IP của nạn nhân hoặc gateway) thì sẽ không khớp binding → gói bị đánh rớt (drop) và không đến được nạn nhân hay gateway.

→ Nhờ vậy, DAI sẽ ngăn chặn hiệu quả ARP Spoofing.

## IV. CẤU HÌNH DHCP SNOOPING
- ```config terminal```
- ```ip dhcp snooping```
- ```ip dhcp snooping vlan 1``` (cài đặt DHCP Snooping cho Vlan 1)
- ```show ip dhcp snooping binding``` (show ra bảng binding)

## V. CẤU HÌNH DYNAMIC ARP INSPECTION
- ```config terminal```
- ```ip arp inspection vlan 1``` (cài đặt DAI cho VLan 1)
- ```ip arp inspection validate src-mac dst-mac ip```
- ```interface GigabitEthernet0/0```(cổng từ switch đến router không cần kiểm tra nên để trust )
- ```ip arp inspection trust ```
- ```end```

- ```no ip arp inspection vlan 1``` (Tắt DAI)
- ```show ip arp inspection interfaces``` (Xem bảng trust/untrust)
- ```show run | s arp``` ( kiểm tra DAI chạy chưa)

## VI. CẤU HÌNH TELNET Ở ROUTER
- ```enable secret cisco```
- ```line vty 0 4```
- ```password 123```
- ```login```

## VII. THỰC HIỆN TẤN CÔNG ARP SPOOFING
- ```sudo sysctl -w net.ipv4.ip_forward=1``` (forward gói tin về lại client)
- ```nmap -sn 192.168.1.0/24``` (liệt kê host đang hoạt động, để tìm ra gateway và nạn nhân )
- ```arpspoof -i eth0 -t 192.168.1.9 -r 192.168.1.1```(trường hợp gateway có IP là 192.168.1.1 và nạn nhân có IP là 192.168.1.9, tiến hành tấn công ARPSpoof )

## VIII. BỔ SUNG
- Nếu có thực hiện tấn công DNS Spoofing kết hợp ARP Spoofing thì ta dựng sẵn web server bằng Apache, Nginx,.. trên máy tấn công và tiến hành cấu hình file /etc/ettercap/etter.dns bằng cách thêm câu lệnh ```*			A	192.168.1.9 ``` với 192.168.1.9 là IP của máy tấn công, * là mọi truy cập website đều sẽ điều hướng về web 192.168.1.9.
- Nếu chỉ muốn khiến nạn nhân mất mạng, không truy cập được internet thì ta có thể thực hiện ```sudo sysctl -w net.ipv4.ip_forward=0```, từ chối việc chuyển tiếp gói tin, khiến gói tin mắc kẹt ở máy kẻ tấn công mà không được chuyển đi tiếp.
- Tấn công nghe lén, ta thực hiện bằng cách ```sudo sysctl -w net.ipv4.ip_forward=1```, cho phép biến máy kẻ tấn công thành trạm trung gian chuyển tiếp gói tin , từ đó thu thâp dữ liệu bằng các công cụ chẳng hạn như WireShark, từ đó lấy được những thông tin được gửi đi qua các kênh truyền không an toàn, không được mã hóa.
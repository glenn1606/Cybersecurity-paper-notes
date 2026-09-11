
**Paper Title:** FEDERAL INFORMATION PROCESSING STANDARDS PUBLICATION 

**Author / Issuing Organization:** National Institute of Standards and Technology (NIST) — Information Technology Laboratory (U.S. Department of Commerce) 

**Publisher / Target:** U.S. Department of Commerce, National Institute of Standards and Technology (NIST)
<<<<<<< HEAD
**Year:** 2015  

\## 1\. Tổng quan &amp; Danh mục Thuật ngữ (Mục 2) ### Các thuật ngữ chính (Mục 2.1) - \*\*Bit / Byte\*\*: Bit là chữ số nhị phân (\`0\` hoặc \`1\`); Byte là chuỗi gồm 8 bit [1]. - \*\*Capacity ($c$)\*\*: Trong mô hình bọt biển (Sponge), dung lượng bằng độ rộng hoán vị trừ đi tốc độ ($c = b - r$) [1, 2]. - \*\*Digest\*\*: Giá trị băm (chuỗi bit đầu ra cố định) của một hàm băm mật mã [1]. - \*\*Domain Separation\*\*: Phân tách miền đầu vào cho các ứng dụng khác nhau để tránh trùng lặp [1, 3, 4]. - \*\*XOF (Extendable-Output Function)\*\*: Hàm đầu ra mở rộng với độ dài tùy chọn theo nhu cầu (gồm SHAKE128 và SHAKE256) [3, 5]. - \*\*Các mảng con của State Array\*\*: - \*\*Column\*\*: Mảng 5 bit với tọa độ $x, z$ cố định [1]. - \*\*Row\*\*: Mảng 5 bit với tọa độ $y, z$ cố định [6]. - \*\*Lane\*\*: Chuỗi $w = b/25$ bit với tọa độ $x, y$ cố định [7, 8]. - \*\*Plane\*\*: Mảng $b/5$ bit với tọa độ $y$ cố định [8, 9]. - \*\*Sheet\*\*: Mảng $b/5$ bit với tọa độ $x$ cố định [6]. - \*\*Slice\*\*: Mảng 25 bit với tọa độ $z$ cố định [6]. ### Biến &amp; Ký hiệu Thuật toán (Mục 2.2 - 2.4) - \*\*$A$\*\*: Mảng trạng thái 3 chiều kích thước $5 \\times 5 \\times w$ [10, 11]. - \*\*$b$\*\*: Độ rộng hoán vị tính bằng bit ($b \\in \\{25, 50, 100, 200, 400, 800, 1600\\}$) [10, 12, 13]. - \*\*$w$\*\*: Kích thước làn tính bằng bit ($w = b/25 \\in \\{1, 2, 4, 8, 16, 32, 64\\}$) [8, 13, 14]. - \*\*$l$\*\*: Logarithm cơ số 2 của kích thước làn ($l = \\log\_2(w)$) [13-15]. - \*\*$n\_r$\*\*: Số vòng lặp của hoán vị $\\text{KECCAK-}p$ [15, 16]. - \*\*Hàm được quy định\*\*: 5 phép biến đổi bước ($\\theta, \\rho, \\pi, \\chi, \\iota$), hoán vị $\\text{KECCAK-}f[b]$, hoán vị tổng quát $\\text{KECCAK-}p[b, n\_r]$, quy tắc độn $pad10^\*1$, các hàm băm SHA-3, các hàm XOF SHAKE và mô hình bọt biển $\\text{SPONGE}$ [16-19]. --- ## 2\. Chi tiết Hoán vị $\\text{KECCAK-}p$ (Chương 3) ### Tổng quan Hoán vị $\\text{KECCAK-}p[b, n\_r]$ (Mục 3) - Hoán vị được xác định bởi độ rộng $b$ ($b \\in \\{25, 50, 100, 200, 400, 800, 1600\\}$) và số vòng $n\_r$ [12, 13, 16]. - Mỗi vòng lặp $\\text{Rnd}$ thực hiện liên tiếp 5 phép biến đổi bước: $\\theta \\to \\rho \\to \\pi \\to \\chi \\to \\iota$ [12, 20, 21]. ### Biểu diễn Trạng thái - State (Mục 3.1) State $b$ bit được biểu diễn song song dưới 2 dạng [11, 13, 22]: 1\. \*\*Chuỗi bit 1D ($S$)\*\*: Độ dài $b$ bit, chỉ số từ $0$ đến $b-1$ [13]. 2\. \*\*Mảng 3D ($A$)\*\*: Kích thước $5 \\times 5 \\times w$ với hệ tọa độ $(x, y, z)$ [11, 22]. - \*\*Chuyển từ $S \\to A$ (Mục 3.1.2)\*\*: Công thức $A[x, y, z] = S[w(5y + x) + z]$ [23]. - \*\*Vai trò của Mảng 3D\*\*: - Đơn giản hóa các công thức toán học cho 5 phép biến đổi bước. - Trực quan hóa đường truyền khuếch tán dữ liệu trong phân tích mật mã. - Tối ưu hóa cài đặt trên máy tính (khi $b=1600$, mỗi làn $w=64$ tương ứng vừa đúng một từ máy 64-bit). ### Chuyển đổi từ Mảng Trạng thái sang Chuỗi Bit (Mục 3.1.3) - \*\*Chuỗi Làn\*\*: $\\text{Lane}(i, j) = A[i, j, 0] \\parallel A[i, j, 1] \\parallel \\dots \\parallel A[i, j, w-1]$ [24]. - \*\*Chuỗi Mặt phẳng\*\*: $\\text{Plane}(j) = \\text{Lane}(0, j) \\parallel \\text{Lane}(1, j) \\parallel \\text{Lane}(2, j) \\parallel \\text{Lane}(3, j) \\parallel \\text{Lane}(4, j)$ [24]. - \*\*Chuỗi Bit hoàn chỉnh $S$\*\*: $S = \\text{Plane}(0) \\parallel \\text{Plane}(1) \\parallel \\text{Plane}(2) \\parallel \\text{Plane}(3) \\parallel \\text{Plane}(4)$ [25]. ### Quy ước Gán nhãn (Mục 3.1.4) - Làn tại tọa độ $(x, y) = (0, 0)$ luôn nằm ở vị trí trung tâm của các lát cắt (Slice) trên các sơ đồ minh họa [26]. ### Các Phép Biến đổi Bước - Step Mappings (Mục 3.2) - Cả 5 phép toán ($\\theta, \\rho, \\pi, \\chi, \\iota$) đều nhận đầu vào là mảng trạng thái $A$ và trả về mảng trạng thái mới $A'$ [27]. - Phép $\\iota$ nhận thêm tham số chỉ số vòng $i\_r$, còn 4 phép biến đổi ($\\theta, \\rho, \\pi, \\chi$) độc lập với chỉ số vòng [28].
=======

**Year:** 2015  
-------------------------------------------------------------------------------------------------------------------------------------------------


**Summary (Introduction & Glossary):**

* **Purpose:** Standardizes the SHA-3 family of functions based on the KECCAK algorithm (winner of the NIST SHA-3 Cryptographic Hash Algorithm Competition) to complement existing SHA-1 and SHA-2 standards.

* **Standardized Functions:**

   4 Cryptographic Hash Functions: SHA3-224, SHA3-256, SHA3-384, and SHA3-512 with fixed output digest lengths.

   2 Extendable-Output Functions (XOFs): SHAKE128 and SHAKE256, allowing arbitrary output lengths tailored to application requirements.

* **Core Architecture:** Built upon the sponge construction using underlying KECCAK-p mathematical permutations.
>>>>>>> 55da7ed30b161ec3e9ca636ccb9e7fd1bb515765

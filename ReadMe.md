# Huớng dẫn đóng gói một ứng dụng dot net core
## Thử ngay mã nguồn
```sh
git clone https://github.com/TechMaster/DockerizeDotNetCoreConsoleApp.git
cd DockerizeDotNetCoreConsoleApp
docker build -t helloworld .
docker images | grep helloworld 
docker run --rm helloworld
```
## Bước 1: Tạo ứng dụng dotnetcore

```sh
dotnet new console -n HelloWorld -o abcxyz
cd abcxyz
dotnet build
dotnet run
```
-n đặt tên cho ứng dụng, nên trùng tên với tên của Docker Image. Tên này cũng dùng cho file định nghĩa dự án *.csproj
-o xuất ra thư mục nào. Nếu chỉ có tên không ví dụ abcxyz thì tạo thư mục abcxyz ở thư mục hiện thời

Nếu không có -o thì thư mục chứa ứng dụng sẽ giống với tên dự án ứng dụng

## Bước 2: Lập trình bằng Visual Studio Code
Khi đã vào trong thư mục dự án gõ lệnh sau:
```
code .
```
*code* là lệnh tắt để khởi động Visual Studio Code từ terminal. Trên MacOSX thường không có sẵn, hãy tham khảo tập hàm Fish Function để tạo lệnh tắt ở đây
[https://github.com/TechMaster/FishFunctionForMac](https://github.com/TechMaster/FishFunctionForMac)

Visual Studio Code sẽ gợi ý bổ xung 2 files trong thư mục .vscode:
- launch.json
- tasks.json

Hai file này cấu hình cho việc biên dịch (build) và gỡ rối (debug). Hãy thử Debug bằng Debug > Start Debugging

## Bước 3: Tạo .dockerignore
Khi lập trình - gỡ rối, 2 thư mục bin và obj sẽ được tạo ra. Khi build docker image cần loại bỏ 2 thư mục này khi COPY. Tạo file .dockerignore có nội dung như sau:
```
bin/
obj/
```
## Bước 4: Tạo Dockerfile cho môi trường build-env
Đóng gói ứng dụng DotNetCore vào Docker image sẽ gồm 2 bước:
1. Biên dịch ứng dụng trong Docker image cài dotnetcore SDK, *FROM microsoft/dotnet:2.0-sdk AS build-env*
2. Kết quả biên dịch bước 1 là file binary được copy sang docker image cài dotnetcore run time *FROM microsoft/dotnet:2.0-runtime*

Trong phần này ta chỉ thực hiện bước 1
```Dockerfile
FROM microsoft/dotnet:2.0-sdk AS build-env      # Docker image chưa dotnetcore SDK
WORKDIR /app                                    # Chọn thư mục mặc định là /app trong Docker image

# copy csproj and restore as distinct layers
COPY *.csproj ./                                # Copy file định nghĩa dự án vào thư mục

RUN dotnet restore                              # Tải về khôi phục các thư viện mà dự án sử dụng

# copy everything else and build
COPY . ./                                       # copy toàn bộ nội dung thư mục dự án vào thư mục mặc định trong images. Cần phải có .dockerignore để loại trừ 2 thư mục bin và obj sinh ra trong lúc biên dịch - gỡ rối khi phát triển
RUN dotnet publish -c Release -o out --no-restore 
# Đóng gói ứng dụng và thư viện liên quan vào thư mục -o out lệnh này sử dụng dotnetcore framework định nghĩa trong file *.csproj
# -c Release dùng cấu hình Release để triển khai lên môi trường sản xuất. Nếu chưa biên dịch thì sẽ biên dịch trong lệnh này
# --no-restore ở trên đã gọi lệnh dotnet restore rồi, nên ở lệnh này không cần restore nữa
```
Chạy lệnh *docker build -t helloworld .* sẽ tạo ra image có kích thước lớn, chứa dotnetcore SDK và toàn bộ source code của dự án.
```sh
docker run --name helloapp -it helloworld:latest
dotnet --info
cd /app/out
dotnet HelloWorld.dll
```
### Thử nghiệm với Dockerfile chỉ dừng ở dotnet publish, chưa copy kết quả vào dotnetcore runtime
```Dockerfile
FROM microsoft/dotnet:2.0-sdk AS build-env
WORKDIR /app

# copy csproj and restore as distinct layers
COPY *.csproj ./

RUN dotnet restore

# copy everything else and build
COPY . ./
RUN dotnet publish -c Release -o out --no-restore
```
## Bước 5: Bổ xung lệnh để xây dựng docker image cho dotnetcore runtime
```Dockerfile
# build runtime image
FROM microsoft/dotnet:2.0-runtime                # Sử dụng docker image dotnetcore runtime nhỏ gọn
WORKDIR /app                                     # Chọn thư mục mặc định là /app trong Docker image
COPY --from=build-env /app/out ./                # Copy kết quả biên dịch trong docker image tạm build-ev ở thư mục /app/out vào thư mục mặc định ở docker image run time

ENTRYPOINT ["dotnet", "HelloWorld.dll"]          # chạy lệnh dotnet HelloWorld.dll
```
## Bước 5: Đóng gói vào Docker image và chạy thử
```sh
docker build -t helloworld .
docker images | grep helloworld 
docker run --rm helloworld
```
# Cloud Computer 
Build data warehouse in operating system linux by docker

Download docker container has built data warehouse : docker pull thuyuong/ubuntu:18.04
-	Link down datayoutube.txt: https://drive.google.com/file/d/1uJ72mxyNLnzapv_-7cJyjgM93nBsW_xh/view?usp=sharing
-	Câu lệnh download file datayoutube.txt:  wget --no-check-certificate 'https://docs.google.com/uc?export=download&id=1uJ72mxyNLnzapv_-7cJyjgM93nBsW_xh' -O datayoutube.txt 
 
 
-	Tạo database: 
Hive> CREATE DATABASE youtube;
 
-	Tạo bảng dwh_yt_ext từ dữ liệu datayoutube.txt:
CREATE EXTERNAL TABLE IF NOT EXISTS dwh_youtube_ext
(ID string,
Channel_Title string,
Published_At string,
Category_Name string,
Duration int,
View_Count int,
Comment_Count int,
Like_Count int,
Country_code string
 )
 ROW FORMAT DELIMITED
 FIELDS TERMINATED BY '\t'
 STORED AS TEXTFILE
 LOCATION '/data';
 
 
*	Khởi tạo Data Mart
-	Bảng youtube.trending_channel đếm số thời gian video, lượt view, lượt like và lượt comment của các Channel_Title thuộc các Category_Name
CREATE TABLE youtube.trending_channel 
AS 
(
SELECT Channel_Tilte,Category_Name,count(Duration) as total_time,count(View_Count) as total_view,count(Like_Count) as total_like,count(Comment_Count) as total_comment
FROM dwh_youtube_ext
GROUP BY Channel_Tilte,Category_Name
);
 
-	Bảng youtube.trending_date có mục đích tính tổng thời gian video, lượt view, lượt like và lượt comment trong từng thời gian Published_At thuộc các Category_Name
CREATE TABLE youtube.trending_date
AS 
(
SELECT to_date(Published_At) as day,Category_Name,sum(Duration) as total_time,sum(View_Count) as total_view,sum(Like_Count) as total_like,sum(Comment_Count) as total_comment
FROM dwh_youtube_ext
GROUP BY to_date(Published_At), Category_Name
);
 
-	Bảng youtube.trending_categories_country có mục đích tính tổng thời gian video, lượt view, lượt like và lượt comment trong từng Country_Code thuộc các Category_Name
CREATE TABLE youtube.trending_categories_country 
AS 
(
SELECT Category_Name,sum(Duration) as total_time,sum(View_Count) as total_view,sum(Like_Count) as total_like,sum(Comment_Count) as total_comment,Country_Code
FROM dwh_youtube_ext
GROUP BY Category_Name, Country_Code
);
 
*	Các câu lệnh truy vấn 
-	Tổng thời gian video, lượt view, lượt like và lượt comment của thể loại video Education vào 10 ngày gần nhất
select day,total_time,total_view,total_like,total_comment
from youtube.trending_date
where Category_Name= 'Education'
group by day,total_time,total_view,total_like,total_comment
order by day DESC limit 10;
 
 
-	Đếm thời gian video, lượt view, lượt like và lượt comment của thể loại Film & Animation ở các nước
select Country_code, trending_categories_country.total_time, trending_categories_country.total_view, trending_categories_country.total_like, trending_categories_country.total_comment
from youtube.trending_channel INNER JOIN youtube.trending_categories_country ON trending_channel.Category_Name=trending_categories_country.Category_Name 
where trending_channel.Category_Name= 'Film & Animation'
Group by Country_code, trending_categories_country.total_time, trending_categories_country.total_view, trending_categories_country.total_like, trending_categories_country.total_comment;
 
 
-	Top 10 tên video và thể loại video được xem nhiều nhất ở nước Mĩ 
select trending_channel.Category_Name,Channel_Title, trending_channel.total_view
from youtube.trending_channel INNER JOIN youtube.trending_categories_country ON trending_channel.Category_Name=trending_categories_country.Category_Name 
where Country_Code = 'US'
group by trending_channel.Category_Name,Channel_Title,trending_channel.total_view
order by trending_channel.total_view DESC limit 10;
 
 
-	10 tên video của thể loại Music vào ngày '2007-10-26' ở nước Mĩ
select b.Channel_Title
from youtube.trending_categories_country LEFT JOIN (select Category_Name,Channel_Title,day from youtube.trending_channel, youtube.trending_date where trending_date.Category_Name = trending_channel.Category_Name group by Channel_Title,day,trending_channel.Category_Name)b
ON trending_categories_country.Category_Name = b.Category_Name 
where day = '2007-10-26' AND Country_Code = 'US' And b.Category_Name = 'Music'
Group by b.Channel_Title
Limit 10;
 

---
title: Nest JS와 S3로 에디터에 등록되는 이미지 관리하기[1편] (작성 중..)
author: Sanghun lee
date: 2021-09-28 11:33:00 +0800
categories: [Blogging, Nestjs, AWS]
tags: [Nestjs, S3, AWS]
math: true
mermaid: true
image:
  src: https://nestjs.com/img/nest-og.png
  width: 850
  height: 585
---

# 커스템 엘리먼츠

# 1. 서론

이미지 저장 -> DONE
에디터에서 이미지 업로드시 s3에 이미지 자동 업로드 및 postId null값으로 이미지 테이블에 데이터 저장
post작성 완료 후 db에 넣을 때 해당 post와 content에 들어있는 링크들을 정규표현식으로 빼와서 iterator를 돌면서 pmk로 postId와 해당 링크들 연결
이미지 관리 -> DONE
-> cron을 통해서 매주 특정 평일 저녁에 db에 저장되어있는 이미지 링크 중
postId null값인 애들을 db에서 삭제
이미지 삭제 -> DONE
-> 해당 postId가 삭제될때 cascade하여 db에 있는 이미지도 같이 삭제되고
-> 해당 포스트에서 Image에 관한 id를 다 받아와서 imageservice에서 bucket key를 위한 array를 생성
-> s3에서 해당 array로 모두 삭제
-> 삭제 후 에러가 없으면 image 테이블에서도 삭제

유저 (프로필이미지), 크루(프로필&배경이미지), brand(프로필&배경이미지)
이것도 동일하게 rest방식으로 file intercepting필요
추가(null값인 경우) & 삭제(x모양) 버튼을 통해서 다르게 가져가면 될 듯

# 2. 방식

S3와 FileIneterceptor 데코레이터를 활용하여 별도의 Controller 구성 및 데이터베이스를 실제로 수정해주어야 하는부분은 별도로 service에서 처리하였다.

크게 업로드에 두가지 유형이 있다.

1. 유저의 프로필 사진, 브랜드의 배경& 프로필 사진과 같이 삭제를하고 등록을 하는 단일성 파일의 경우 DB에 굳이 등록되지않고 유저 컬럼내에서 관리해도 될거라 판단되었다.
   아래와 같은 방식으로 진행하였다.

- 등록과 동시에 s3에 집어넣고 프론트에서 저장버튼 클릭 시 해당 링크를 가지고 DB에 해당 컬럼에 가지고 있는 방식
- 재등록 또는 수정의 경우 선삭제가 이루어지도록 만들어 삭제버튼 클릭시 해당 링크에서 objectName을 도출하여 DB에서 삭제와 동시에 S3에서 삭제처리

2.  프론트에서 Tui 에디터에 해당 포스트에 들어가는 사진들은 업로드와 동시에 미리 S3에 등록이 되고 Nest JS로 구현된 서버의 Image 테이블에 postId가 null값을 등록이 된다.

- 포스트를 등록하는 순간 해당포스트의 contents내에서 정규표현식을 통해 해당링크들을 긁어오는 배열을 만들고 Image 테이블에서 해당하는 링크를 찾아 post와 관계를 만들어준다.
- 만약 포스트 업로드중 중도 나가는 경우와 매칭되지 않는 사진의 경우 postId는 null값으로 존재하게 되므로 이는 CRON을 통해 매 새벽마다 지워주는 방식을 택했다.

```ts
@Post('')
  @UseInterceptors(FileInterceptor('file'))
  async uploadFile(@UploadedFile() file: any) {
    AWS.config.update({
      credentials: {
        accessKeyId: this.configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: this.configService.get('AWS_SECRET_ACCESS_KEY'),
      },
    });

    try {
      //아래는 버킷생성을 위한 코드이므로 한번 사용 후 주석처리
      // const upload = await new AWS.S3()
      //   .createBucket({
      //     Bucket: BUCKET_NAME,
      //   })
      //   .promise();
      const objectName = `${uuidv4() + file.originalname}`;
      await new AWS.S3()
        .putObject({
          Body: file?.buffer, //파일 내용임
          Bucket: BUCKET_NAME,
          Key: objectName, //파일 이름으로 설정(unique를 위함)
          ACL: 'public-read', //권한변경 설정으로 기본적으로 access denied가 되기에 설정 필요
        })
        .promise();

      //db에 Obj이름을 넣으면 삭제시 더 간단해질 것 같은데 용량이 걱정..
      const url = `https://${BUCKET_NAME}.s3.amazonaws.com/${objectName}`;
      this.imageService.savePostImage({
        link: url,
      });
      return { url };
    } catch (error) {
      console.log(error);
      return null;
    }
  }

```

# 3. 결론

conclusion

## 참고

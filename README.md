# PNU Blace | 스터디마다 다른 출석 기준을 판정하는 코드

박준이 · 1인 개발 및 운영

부산대학교 도서관의 열람실 현황과 스터디 출결을 제공한 개인 프로젝트입니다. 2026년 4월경 도서관과 협의해 운영을 종료했습니다. 당시 앱에서 열람실을 고르고 스터디별 출결 기준을 살펴보는 화면을 담았습니다. [18초 화면 기록](https://zune2222.github.io/pnu-blace-showcase/screens.mp4)

스터디장으로서 매일 참가자들의 출석 인증을 확인하고 지각과 결석을 기록했습니다. 다른 스터디장들도 같은 일을 반복했지만 입실 마감 시각과 의무 학습 시간은 그룹마다 달랐습니다. 그래서 스터디장이 기준을 정하면 서버가 하루의 도서관 이용 기록을 읽어 그룹별 출결을 계산하도록 만들었습니다.

원본은 `apps/api/src/study/attendance.service.ts`이며 행 번호는 커밋 `a68646aa0dcfdcb75df2daf56ac014c4b59a348c` 기준입니다. 제가 작성한 `599c9dc`와 `b903309`의 변경을 포함합니다.

## 하루에 여러 번 입실한 경우

한 학생이 같은 날 두 번 다녀갔다면 첫 입실과 마지막 퇴실은 마감 시각을 판단하는 데 쓰고, 두 방문 사이의 공백은 이용 시간에서 뺐습니다. 당일 기록이 없으면 낮 동안에는 판정을 보류합니다. 별도의 자정 작업이 전날의 그룹 운영일과 휴가 여부를 확인해 결석을 처리합니다.

`syncMemberAttendance`의 168~186행을 발췌했습니다. 앞의 운영일·휴가 확인과 기록 조회, 뒤의 결과 저장은 생략했습니다.

```typescript
      if (libraryRecords.length === 0) {
        // 기록 없음 - 아직 출근 안함 (결석 처리는 자정에)
        return;
      }

      // 첫 입실 시간, 마지막 퇴실 시간 계산
      const checkInTime = this.getEarliestTime(libraryRecords, 'startTm');
      const checkOutTime = this.getLatestTime(libraryRecords, 'endTm');

      // 총 이용 시간 계산
      const usageMinutes = this.calculateTotalUsageMinutes(libraryRecords);

      // 출퇴근 상태 판별
      const status = this.determineAttendanceStatus(
        group,
        checkInTime,
        checkOutTime,
        usageMinutes,
      );
```

`calculateTotalUsageMinutes`의 470~482행입니다. 각 이용 구간의 길이를 더하고, 한 구간 안에서 자정을 넘긴 경우 퇴실 시각에 하루를 보탭니다.

```typescript
  private calculateTotalUsageMinutes(records: LibraryUsageRecord[]): number {
    return records.reduce((total, record) => {
      const startMinutes = this.timeToMinutes(record.startTm);
      let endMinutes = this.timeToMinutes(record.endTm);

      // 자정 넘김 처리
      if (endMinutes < startMinutes) {
        endMinutes += 24 * 60;
      }

      return total + (endMinutes - startMinutes);
    }, 0);
  }
```

## 같은 기록도 스터디에 따라 달라지는 이유

입실 마감과 정확히 같은 시각에 도착하면 정시입니다. 제시간에 도착했어도 최소 퇴실 시각 전에 나가거나 합산 이용 시간이 부족하면 조퇴입니다. 이 구현은 날짜별로 상태를 하나만 저장하므로 지각과 조퇴가 겹치면 지각으로 남습니다.

`determineAttendanceStatus`의 262~294행 전체입니다. 두 시각과 최소 이용 시간은 해당 스터디의 설정에서 가져옵니다.

```typescript
  private determineAttendanceStatus(
    group: StudyGroup,
    checkInTime: string,
    checkOutTime: string | null,
    usageMinutes: number,
  ): AttendanceStatus {
    const checkInMinutes = this.timeToMinutes(checkInTime);
    const checkInEndMinutes = this.timeToMinutes(group.checkInEndTime);
    const checkOutMinMinutes = this.timeToMinutes(group.checkOutMinTime);

    // 지각 체크: 출근 마감 시간 이후 입실
    const isLate = checkInMinutes > checkInEndMinutes;

    // 조퇴 체크: 퇴근 시간 전 퇴실 또는 최소 이용시간 미달
    let isEarlyLeave = false;
    if (checkOutTime) {
      const checkOutMinutes = this.timeToMinutes(checkOutTime);
      isEarlyLeave =
        checkOutMinutes < checkOutMinMinutes ||
        usageMinutes < group.minUsageMinutes;
    }

    // 상태 결정
    if (isLate && isEarlyLeave) {
      return 'LATE'; // 지각 우선
    } else if (isLate) {
      return 'LATE';
    } else if (isEarlyLeave) {
      return 'EARLY_LEAVE';
    }

    return 'PRESENT';
  }
```

## 예시 입력으로 확인한 판정

스터디 A의 입실 마감은 09:00, 최소 퇴실은 11:00, 최소 이용 시간은 120분입니다. 스터디 B는 입실 마감만 08:50입니다. 아래 시각은 검증을 위해 만든 예시입니다.

| 스터디 | 하루 이용 구간 | 합산 | 코드가 반환한 결과 |
| --- | --- | ---: | --- |
| A | 09:00~11:00 | 120분 | 출석 |
| B | 09:00~11:00 | 120분 | 지각 |
| A | 09:00~09:30, 10:30~11:00 | 60분 | 조퇴 |
| A | 08:00~10:30 | 150분 | 조퇴 |
| A | 09:10~10:00 | 50분 | 지각 |

위 버전의 집계·시각 변환 메서드(450~487행)와 판정 메서드(262~294행)를 그대로 불러 Node.js에서 실행했습니다. 다섯 입력 각각에 대해 합산 시간과 최종 상태가 표의 값과 일치하는지 확인했습니다. 이 검증은 도서관 이용 구간이 이미 입력된 상태에서 시작합니다.

<div
  class="clock-form"
  style="display: flex; flex-direction: column; justify-content: center"
>
  <h2>補打卡</h2>

  <div class="form-row">
    <label>員工編號：</label>
    <select [(ngModel)]="clock.employeeId" (ngModelChange)="checkWorkOfDate()">
      @for(item of employeeList;track $index){
      <option [value]="item.id">{{item.id}} - {{item.name}}</option>
      }
    </select>
  </div>
  <!-- ?##################### -->

  <div class="form-row">
    <label>打卡日期：</label>
    <input
      type="date"
      [(ngModel)]="clock.workDate"
      (ngModelChange)="checkRecData()"
    />
  </div>
  <!-- ?##################### -->
  <!-- 有無排班 -->
  @if(shiftWorkOfDate.length != 0) {
  <!-- ? -->
  <!-- 是否部分忘打 -->
  @if (standbyReClockList.length != 0) {
  <p style="margin-top: 0; text-align: center; color: rgb(201, 142, 61)">
    當日尚未補打卡筆數：{{standbyReClockList.length}}
  </p>
  <p style="margin-top: 0; text-align: center; color: rgb(103, 168, 94)">
    待補打卡內容
  </p>

  <div class="form-row">
    <label>上班打卡：</label>
    <input
      type="time"
      step="1"
      name="clockOn"
      disabled
      [(ngModel)]="clock.clockOn"
    />
  </div>

  <div class="form-row">
    <label>下班打卡：</label>
    <input
      type="time"
      step="1"
      [(ngModel)]="clock.clockOff"
      (ngModelChange)="checkClockOff()"
    />
  </div>

  @if(isContinue) {

  <div class="form-row">
    <label>休息開始：</label>
    <input type="time" step="1" [(ngModel)]="clock.restStart" />
  </div>

  <div class="form-row">
    <label>休息結束：</label>
    <input type="time" step="1" [(ngModel)]="clock.restEnd" />
  </div>
  }

  <div class="form-row">
    <label>心情評分：</label>
    <select [(ngModel)]="clock.score">
      <option [value]="0">0</option>
      <option [value]="1">1</option>
      <option [value]="2">2</option>
      <option [value]="3">3</option>
      <option [value]="4">4</option>
      <option [value]="5">5</option>
    </select>
  </div>

  <div style="display: flex; justify-content: center">
    <div class="form-actions" style="margin-right: 4px; margin-top: 0">
      <button (click)="fixClock()">更新</button>
    </div>

    <!-- <div class="form-actions" (click)="submitReqClock()">
         <button>新增</button>
       </div> -->
  </div>

  } @else if (isAllDayForgot) {

  <p style="margin-top: 0; text-align: center; color: rgb(201, 142, 61)">
    當日尚未補打卡筆數：{{checkForgotNum}}
  </p>
  <p style="margin-top: 0; text-align: center; color: rgb(103, 168, 94)">
    待補打卡內容
  </p>

  <!-- ! -->
  <div class="form-row">
    <label>上班打卡：</label>
    <input type="time" step="1" name="clockOn" [(ngModel)]="clock.clockOn" />
  </div>

  <div class="form-row">
    <label>下班打卡：</label>
    <input
      type="time"
      step="1"
      [(ngModel)]="clock.clockOff"
      (ngModelChange)="checkClockOff()"
    />
  </div>

  @if(isContinue) {

  <div class="form-row">
    <label>休息開始：</label>
    <input type="time" step="1" [(ngModel)]="clock.restStart" />
  </div>

  <div class="form-row">
    <label>休息結束：</label>
    <input type="time" step="1" [(ngModel)]="clock.restEnd" />
  </div>
  }

  <div class="form-row">
    <label>心情評分：</label>
    <select [(ngModel)]="clock.score">
      <option [value]="0">0</option>
      <option [value]="1">1</option>
      <option [value]="2">2</option>
      <option [value]="3">3</option>
      <option [value]="4">4</option>
      <option [value]="5">5</option>
    </select>
  </div>

  <div style="display: flex; justify-content: center">
    @if(isAllDayForgot) {

    <div class="form-actions" style="margin-right: 4px; margin-top: 0">
      <button (click)="fixClock()">更新資料</button>
    </div>
    }

    <!-- <div class="form-actions" (click)="submitReqClock()">
         <button>新增</button>
       </div> -->
  </div>
  <!-- ! -->

  } @else if (clock.workDate != '') {
  <span style="text-align: center; color: rgb(192, 131, 131)"
    >查無不完整打卡紀錄</span
  >

  }
  <!-- ? -->
  } @else {
  <span style="text-align: center; color: rgb(85, 98, 210)">當日未排班</span>
  } @if(clock.workDate == '') {

  <span style="text-align: center; color: rgb(89, 118, 198)"
    >請選擇職員編號與補打卡日期</span
  >
  }
</div>






import { Component } from '@angular/core';
import { MatDialog, MatDialogRef } from '@angular/material/dialog';
import { FormsModule } from '@angular/forms';
import { HttpClientService } from '../../../@Service/HttpClientService';
import { Success } from '../../success/success';
import { Fail } from '../../fail/fail';

@Component({
  selector: 'app-punch-in-late',
  imports: [FormsModule],
  templateUrl: './punch-in-late.html',
  styleUrl: './punch-in-late.scss',
})
export class PunchInLate {
  //建構式
  constructor(
    private http: HttpClientService,
    private dialog: MatDialog,
    private dialogRef: MatDialogRef<PunchInLate>
  ) {}

  //初始化
  ngOnInit(): void {
    //取得在職員工
    this.http
      .getApi(`http://localhost:8080/head/searchAllNotResign`)
      .subscribe((res: any) => {
        this.employeeList = res.searchResList;
        console.log(res);
      });

    console.log(this.shiftWorkOfDate);
    console.log(this.standbyReClockList);
  }

  //全域變數
  employeeList: any[] = [];
  check = false;
  clock: any = {
    employeeId: '',
    workDate: '',
    clockOn: '',
    clockOff: '',
    restStart: '',
    restEnd: '',
    score: '',
  };

  shiftWorkOfMonth: any[] = []; // 整月已排班
  shiftWorkOfDate: any[] = []; // 當日已排班
  standbyReClockList: any[] = []; // 待補打卡列表
  isContinue: any; // 是否連續
  isAllDayForgot: boolean = false; // 全天忘打？
  allDayReClockNum: any;
  checkForgotNum: number = 0;

  checkRecData() {
    this.standbyReClockList = [];
    this.clock.score = null;

    this.http
      .getApi(
        `http://localhost:8080/single/date?employeeId=${this.clock.employeeId}&workDate=${this.clock.workDate}`
      )
      .subscribe((res: any) => {
        // console.log('當日打卡資訊', res); // 當日打卡資訊

        // ? =============== 鬼畫符開始 ===============
        const scheduleToday = (this.shiftWorkOfMonth ?? []).filter(
          (i: any) => i.applyDate == this.clock.workDate
        );
        const scheduleCount = scheduleToday.length;

        const isContinueLocal =
          scheduleCount === 2 &&
          Math.abs(
            Number(scheduleToday[1]?.shiftWorkId ?? 0) -
              Number(scheduleToday[0]?.shiftWorkId ?? 0)
          ) === 1;

        const punchesCountAll = Array.isArray(res?.data) ? res.data.length : 0;
        const partialMissing =
          Array.isArray(res?.data) &&
          res.data.some((p: any) => p.clockOff == null);

        if (res?.data == null) {
          this.checkForgotNum = isContinueLocal ? 1 : scheduleCount;
        } else {
          const punchesCount = Array.isArray(res.data) ? res.data.length : 0;
          this.checkForgotNum = Math.max(0, scheduleCount - punchesCount);
        }

        const scheduleTodaySorted = [...scheduleToday].sort(
          (a: any, b: any) => Number(a.shiftWorkId) - Number(b.shiftWorkId)
        );
        const firstShift = scheduleTodaySorted[0];
        const secondShift = scheduleTodaySorted[1];

        const spansTwoShifts =
          isContinueLocal && !partialMissing && punchesCountAll >= 1;

        // 覆蓋的班別數（若跨兩班，視 1 筆覆蓋 2 班）
        const effectivePunches = spansTwoShifts
          ? Math.max(2, punchesCountAll)
          : punchesCountAll;

        if (Array.isArray(res?.data) && spansTwoShifts) {
          this.checkForgotNum = Math.max(0, scheduleCount - 2);
        }

        console.log(
          'checkForgotNum =',
          this.checkForgotNum,
          ' scheduleCount=',
          scheduleCount,
          ' isContinueLocal=',
          isContinueLocal
        );
        // ? =============== 鬼畫符終了 ===============

        if (res.data != null) {
          this.isAllDayForgot = false;
          res.data.forEach((i: any) => {
            if (i.clockOff == null) {
              this.standbyReClockList.push(i);
            }
          });
          console.log('等待補打卡列表', this.standbyReClockList);
          if (this.standbyReClockList.length != 0) {
            this.clock.clockOn = this.standbyReClockList[0].clockOn;
            console.log(
              '準備補打卡資料 standbyReClockList',
              this.standbyReClockList[0]
            );
          } else {
            this.clock.clockOn = '';
          }
          this.shiftWorkOfDate = this.shiftWorkOfMonth.filter((i) => {
            return i.applyDate == this.clock.workDate;
          });

          if (
            this.shiftWorkOfDate.length == 2 &&
            Math.abs(
              this.shiftWorkOfDate[1].shiftWorkId -
                this.shiftWorkOfDate[0].shiftWorkId
            ) == 1
          ) {
            this.isContinue = true;
          } else {
            this.isContinue = false;
          }

          if (scheduleCount - punchesCountAll > 0 && !partialMissing) {
            this.isAllDayForgot = true;
          }

          if (spansTwoShifts) {
            this.isAllDayForgot = false;
          }

          console.log(
            `當日已排班資訊，共 ${this.shiftWorkOfDate.length} 個班別(${
              this.isContinue ? '連續' : '非連續'
            })`,
            this.shiftWorkOfDate
          );
          console.log('應打卡 SHIFT_WORK_OF_DATE', this.shiftWorkOfDate);
          console.log('當日打卡狀況 RES', res);
          console.log('是否整天未打卡', this.isAllDayForgot);
        } else if (res.data == null) {
          this.isAllDayForgot = true;
          this.clock.clockOn = '';

          // console.log('SHIFT_WORK_OF_MONTH', this.shiftWorkOfMonth);
          console.log('選擇日期', this.clock.workDate);

          this.shiftWorkOfDate = this.shiftWorkOfMonth.filter((i) => {
            return i.applyDate == this.clock.workDate;
          });

          if (this.shiftWorkOfDate.length == 0) {
            this.isAllDayForgot = false;
          }

          if (
            this.shiftWorkOfDate.length == 2 &&
            Math.abs(
              this.shiftWorkOfDate[1].shiftWorkId -
                this.shiftWorkOfDate[0].shiftWorkId
            ) == 1
          ) {
            this.isContinue = true;
          } else {
            this.isContinue = false;
          }

          this.allDayReClockNum += this.shiftWorkOfDate.length;

          console.log('應打卡 SHIFT_WORK_OF_DATE', this.shiftWorkOfDate);
          console.log('當日打卡狀況 RES', res);
          console.log('是否整天未打卡', this.isAllDayForgot);
        }
        // else {
        //   console.log('查無需要補打卡內容');
        // }
      });
    console.log('standbyReClockList', this.standbyReClockList);
    console.log('CLOCK', this.clock);
  }

  // 取得員工當月有上班的所有日期
  checkWorkOfDate() {
    this.shiftWorkOfMonth = [];
    this.standbyReClockList = [];
    this.clock.workDate = '';

    this.shiftWorkOfDate = [];
    this.isAllDayForgot = false;
    this.isContinue = false;
    this.checkForgotNum = 0;
    this.http
      .getApi(
        `http://localhost:8080/PreSchedule/getAcceptScheduleByEmployeeId?employeeId=${this.clock.employeeId}`
      )
      .subscribe((res: any) => {
        this.shiftWorkOfMonth = res.preScheduleList.filter((i: any) => {
          return i.shiftWorkId != 0;
        });
        // console.log('當月預排班表', res.preScheduleList);
        // console.log(
        //   '當月有上班日期 SHIFT_WORK_OF_MONTH',
        //   this.shiftWorkOfMonth
        // );
      });

    // console.log('clock', this.clock);
  }

  checkClockOff() {
    console.log(this.clock);
  }

  // 先檢查該員工選取日期的打卡;
  checkFixClock(id: string) {
    if (!id) {
      this.dialog.open(Fail, {
        width: '150px',
        data: { message: '員工 ID 為空' },
      });
      return;
    }
    this.http
      .postApi(`http://localhost:8080/clock/fix/check`, this.clock)
      .subscribe((res: any) => {
        switch (res.status) {
          case 'MISS_OFF':
            this.clock = {
              employeeId: this.clock.employeeId,
              workDate: this.clock.workDate,
              clockOn: this.clock.clockOn,
              clockOff: this.clock.clockOff,
              restStart: this.clock.restStart,
              restEnd: this.clock.restEnd,
              score: this.clock.score,
            };
            this.check = false;
            alert(res.message);
            break;
          case 'MISS_ON':
            this.clock = {
              employeeId: this.clock.employeeId,
              workDate: this.clock.workDate,
              clockOn: this.clock.clockOn,
              clockOff: '',
              score: 0,
            };
            this.check = false;
            alert(res.message);
            break;
          case 'MISS_TWO':
            this.clock = {
              employeeId: this.clock.employeeId,
              workDate: this.clock.workDate,
              clockOn: this.clock.clockOn,
              clockOff: this.clock.clockOff,
              score: 0,
            };
            this.check = true;
            alert(res.message);
            break;
          default:
            return res.message;
        }
      });
  }

  submitReqClock() {
    this.http
      .postApi(`http://localhost:8080/rec/part`, this.clock)
      .subscribe((res: any) => {
        console.log(res);
      });
  }

  //送出
  fixClock() {
    if (!this.check) {
      this.http
        .postApi(`http://localhost:8080/rec/part`, this.clock)
        .subscribe((res: any) => {
          if (res.code == 200) {
            this.dialog.open(Success, { width: '150px' });
            this.dialogRef.close(true);
          } else {
            this.dialog.open(Fail, {
              width: '150px',
              data: { message: res.message },
            });
            return;
          }
        });
    } else {
      this.http
        .postApi(`http://localhost:8080/clock/fix/create`, this.clock)
        .subscribe((res: any) => {
          if (res.code == 200) {
            this.dialog.open(Success, { width: '150px' });
            this.dialogRef.close(true);
          } else {
            this.dialog.open(Fail, {
              width: '150px',
              data: { message: res.message },
            });
            return;
          }
        });
    }
  }
}

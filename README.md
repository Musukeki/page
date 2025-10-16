// ClockServiceImpl.java（重點流程）
package com.example.p01.service.impl;

import java.time.Duration;
import java.time.LocalDate;
import java.time.LocalTime;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.stream.Collectors;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.example.p01.dao.ClockDateDao;
import com.example.p01.dao.ClockDateMyBatisDao;
import com.example.p01.dao.HolidayDao;
import com.example.p01.dto.ClockDateDto;
import com.example.p01.dto.ClockOffDto;
import com.example.p01.dto.response.ApiRes;
import com.example.p01.entity.ClockDate;
import com.example.p01.service.ifs.ClockDateService;
import com.example.p01.vo.ClockDateVo.*;
import com.example.p01.vo.headVo.BasicRes;

import jakarta.transaction.Transactional;

@Service
public class ClockServiceImpl implements ClockDateService {

	@Autowired
	private ClockDateDao clockDateDao;

	@Autowired
	private HolidayDao holidayDao;

	@Autowired
	private ClockDateMyBatisDao clockDateMyBatisDao;



	@Override
	public BasicRes clockOff2(ClockOffDto dto) throws Exception {

		try {
			// 取該員工的最新一筆資料出來
			ClockDateDto lastData = clockDateMyBatisDao.selectClockDate(dto.getEmployeeId());

			// 判斷下班時間有沒有跨日
			// 如果有跨日 日期減一
			LocalDate checkDate;
			if (dto.getClockOff().isAfter(lastData.getClockOn())) {

				checkDate = LocalDate.now();

			} else {

				checkDate = LocalDate.now().minusDays(1);

			}

			// 檢查打下班卡時，有沒有先打上班的卡

			// 取最新的一筆資料出來 檢查上班打卡日期有沒有符合當下時間
//			if(lastData.getWorkDate()!=checkDate) {
//				return new BasicRes(400, "資料庫找不到符合的資料，請先確認是否有忘記打卡");
//			}

			ClockOffDto anw = new ClockOffDto(dto.getClockOff(), //
					dto.getScore(), dto.getEmployeeId());

			// 打下班卡紀錄時間 只更新最新的一筆資料
			clockDateMyBatisDao.clockOff(anw);

			// 更新後重新查一次最新資料
			lastData = clockDateMyBatisDao.selectClockDate(dto.getEmployeeId());

			double hours;
			// LocalTime.MAX = 23:59:59.999999999
			// LocalTime.MIDNIGHT = 00:00
			if (lastData.getClockOff().isBefore(lastData.getClockOn())) {
				hours = Duration.between(lastData.getClockOn(), LocalTime.MAX).toMinutes()
						+ Duration.between(LocalTime.MIDNIGHT, lastData.getClockOff()).toMinutes();
			} else {
				hours = Duration.between(lastData.getClockOn(), lastData.getClockOff()).toMinutes();
			}

			// 計算暫離的時間 
			double leaveTime;
			if (lastData.getRestStart() != null && lastData.getRestEnd() != null) {
				
				 leaveTime = Duration.between(lastData.getRestStart(), //
						lastData.getRestEnd()).toMinutes();
				 
				 hours = hours-leaveTime;
			}

			// 向下取整數 例: 2.7->2 -2.7->-3
			hours = Math.floor(hours / 60.0 * 2) / 2.0;

			anw.setTotalHour(hours);
			// 計算完工時，返回資料庫
			clockDateMyBatisDao.clockOff(anw);

		} catch (Exception e) {
			throw e;
		}

		return new BasicRes(200, "success");
	}
	
	// ================================================================================
	
	// 查詢個別員工當日打卡紀錄
    @Override
    public ApiRes<List<ClockDate>> getSingleClock(String employeeId, LocalDate workDate) {

        List<ClockDate> result = clockDateDao.getSingleClock(employeeId, workDate);

        if (result.isEmpty()) {
            return new ApiRes<>(400, "查無打卡資料", null);
        }

        for (ClockDate cd : result) {

            Integer cls = clockDateDao.findShiftClass(cd.getEmployeeId(), cd.getWorkDate(), cd.getClockOn());
            cd.setClazz(cls);
        }

        return new ApiRes<>(200, "success", result);
    }

    // 查詢所有員工歷史打卡紀錄
    @Override
    public ApiRes<List<ClockDate>> getAllClock() {

        List<ClockDate> result = clockDateDao.getAllClock();

        return new ApiRes<>(200, "success", result);
    }

    // 查詢個別員工歷史打卡紀錄
    @Override
    public ApiRes<List<ClockDate>> getSingleHistoryClock(String employeeId) {

        List<ClockDate> result = clockDateDao.getSingleHistoryClock(employeeId);

        if (result.isEmpty()) {
            return new ApiRes<>(400, "查無打卡資料", null);
        }
        return new ApiRes<>(200, "success", result);
    }

    // 上班打卡
    @Transactional
    @Override
    public ApiRes<ClockDate> addClockOn(AddClockOnReq req) {

        boolean hasPre = clockDateDao.checkAcceptPre(req.getEmployeeId(), req.getWorkDate(), req.getClockOn()) == 1;

        if (!hasPre) {
            return new ApiRes<>(400, "該時段未排班", null);
        }

        // boolean hasExistsOn = dao.checkExistsOn(req.getEmployeeId(),
        // req.getWorkDate(), req.getClockOn()) == 1;

        boolean isContinuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;
        if (isContinuous) {
            boolean hasAnyOnToday = clockDateDao.hasClockOn(req.getEmployeeId(), req.getWorkDate()) == 1;
            if (hasAnyOnToday) {
                return new ApiRes<>(400, "該時段已有打卡紀錄", null);
            }
        } else {
            // 非連續班同一班別不可重複
            boolean hasExistsOn = clockDateDao.checkExistsOn2(
                    req.getEmployeeId(), req.getWorkDate(), req.getClockOn()) == 1;
            if (hasExistsOn) {
                return new ApiRes<>(400, "該時段已有打卡紀錄", null);
            }
        }

        int changed = clockDateDao.addClockOnWhenNotContinuous(req.getEmployeeId(), req.getWorkDate(), req.getClockOn());

        if (changed == 1) {

            boolean continuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;

            if (continuous) {
                return new ApiRes<>(200, "打卡成功", null);
            } else {
                return new ApiRes<>(200, "打卡成功", null);
            }

        } else {

            return new ApiRes<>(500, "未知錯誤", null);
        }

    }

    // 下班打卡
    @Transactional
    @Override
    public ApiRes<ClockDate> addClockOff(AddClockOffReq req) {

        return null;
    }

    // 休息開始
    @Transactional
    @Override
    public ApiRes<ClockDate> addRestStart(AddRestStartReq req) {

        boolean continuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;

        if (!continuous) {
            return new ApiRes<>(200, "非連續時段班無法休息", null);
        }

        boolean checkRangeOfRest = clockDateDao.canStartRest(req.getEmployeeId(), req.getWorkDate(), req.getRestStart()) == 1;

        if (!checkRangeOfRest) {
            return new ApiRes<>(400, "未在上班時段之間打卡", null);
        }

        boolean hasRestStart = clockDateDao.existsRestStart(req.getEmployeeId(), req.getWorkDate(), req.getRestStart()) == 1;

        if (hasRestStart) {
            return new ApiRes<>(400, "已有休息開始紀錄", null);
        }

        boolean row = clockDateDao.addRestStart(req.getEmployeeId(), req.getWorkDate(), req.getRestStart()) == 1;

        if (!row) {
            return new ApiRes<>(400, "找不到對應的上班打卡紀錄", null);
        }

        return new ApiRes<>(200, "休息開始", null);
    }

    // 休息結束
    @Transactional
    @Override
    public ApiRes<ClockDate> addRestEnd(AddRestEndReq req) {

        boolean continuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;

        if (!continuous) {
            return new ApiRes<>(200, "非連續時段班無法休息", null);
        }

        boolean checkHasOn = clockDateDao.hasClockOn(req.getEmployeeId(), req.getWorkDate()) == 1;

        if (!checkHasOn) {
            return new ApiRes<>(400, "找不到對應的上班打卡紀錄", null);
        }

        boolean checkRestStart = clockDateDao.hasRestStart(req.getEmployeeId(), req.getWorkDate()) == 1;

        if (!checkRestStart) {
            return new ApiRes<>(400, "尚未打休息開始，請補打卡", null);
        }

        boolean hasRestEnd = clockDateDao.existsRestEnd(req.getEmployeeId(), req.getWorkDate(), req.getRestEnd()) == 1;

        if (hasRestEnd) {
            return new ApiRes<>(400, "已有休息結束紀錄", null);
        }

        boolean checkRangeOfRest = clockDateDao.canEndRest(req.getEmployeeId(),
                req.getWorkDate(), req.getRestEnd()) == 1;

        if (!checkRangeOfRest) {
            return new ApiRes<>(400, "未在上班時段之間打卡", null);
        }

        boolean row = clockDateDao.addRestEnd(req.getEmployeeId(), req.getWorkDate(),
                req.getRestEnd()) == 1;

        if (row) {
            return new ApiRes<>(200, "休息結束", null);
        }

        return new ApiRes<>(500, "未知錯誤", null);
    }

    // 補打卡(新增)
    @Transactional
    @Override
    public ApiRes<ClockDate> reClockAll(ReClockReq req) {

        // 判斷打卡資訊是否完整
        if ((req.getClockOn() == null && req.getClockOff() != null) ||
                (req.getClockOn() != null && req.getClockOff() == null) ||
                (req.getClockOn() == null && req.getClockOff() == null)) {
            return new ApiRes<>(400, "請輸入完整打卡資訊", null);
        }

        // 檢查是否有排班
        boolean hasPre = clockDateDao.checkAcceptPre(req.getEmployeeId(), req.getWorkDate(), req.getClockOn()) == 1;
        if (!hasPre) {
            return new ApiRes<>(400, "該時段未排班", null);
        }

        // 判斷是否有休息
        boolean hasRest = (req.getRestStart() != null && req.getRestEnd() != null);

        // 判斷非連續禁止休息
        boolean continuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;
        if (!continuous && hasRest) {
            return new ApiRes<>(200, "非連續時段班無法休息", null);
        }

        // 判斷休息資訊是否完整
        if ((req.getRestStart() == null && req.getRestEnd() != null) ||
                (req.getRestStart() != null && req.getRestEnd() == null)) {
            return new ApiRes<>(400, "請輸入完整休息資訊", null);
        }
        // 只有有給休息才檢查休息是否在上班時段
        if (hasRest) {
            boolean checkRangeOfRest = clockDateDao.canStartRest(req.getEmployeeId(), req.getWorkDate(),
                    req.getRestStart()) == 1;
            if (!checkRangeOfRest) {
                return new ApiRes<>(400, "休息必須在上班時段之間", null);
            }

            if (req.getRestEnd().isBefore(req.getRestStart())) {
                return new ApiRes<>(400, "休息結束不能早於休息開始", null);
            }
        }

        int restTime = 0;
        if (req.getRestStart() != null && req.getRestEnd() != null) {
            restTime = (req.getRestEnd().toSecondOfDay() - req.getRestStart().toSecondOfDay() + 86400) % 86400;
        }

        Double totalHour = Math.max(
                0,
                (req.getClockOff().toSecondOfDay() - req.getClockOn().toSecondOfDay() + 86400) % 86400
                        - restTime)
                / 3600.0;

        totalHour = Math.floor(totalHour * 2.0 + 1e-9) / 2.0;

        int changed = clockDateDao.reClockAll(
                req.getEmployeeId(), req.getWorkDate(),
                req.getClockOn(), req.getClockOff(),
                req.getRestStart(), req.getRestEnd(),
                totalHour,
                req.getScore());

        if (changed >= 1) {
            return new ApiRes<>(200, "補打卡成功", null);
        }
        return new ApiRes<>(500, "未知錯誤", null);
    }

    // 補打卡(更新)
    @Transactional
    @Override
    public ApiRes<ClockDate> reClockPart(ReClockReq req) {

        // 輸入正確資訊
        if ((req.getEmployeeId() == null && req.getWorkDate() != null) ||
                (req.getEmployeeId() != null && req.getWorkDate() == null) ||
                (req.getEmployeeId() == null && req.getWorkDate() == null)) {
            return new ApiRes<>(400, "請輸入正確員工編號及補卡日期", null);
        }

        if (req.getClockOn() == null || req.getClockOff() == null) {
            return new ApiRes<>(400, "請輸入完整打卡資訊", null);
        }

        // 檢查是否有上班打卡
        int checkExists = clockDateDao.checkReClockExists(req.getEmployeeId(), req.getWorkDate(), req.getClockOn());
        if (checkExists == 0) {
            return new ApiRes<>(200, "查無該筆上班打卡紀錄，請確認上班打卡時間是否無誤，或新增(補)完整打卡資訊", null);
        }

        // 判斷是否有休息
        boolean hasRest = (req.getRestStart() != null && req.getRestEnd() != null);

        // 判斷非連續禁止休息
        boolean continuous = clockDateDao.hasContinuousShift(req.getEmployeeId(), req.getWorkDate()) == 1;
        if (!continuous && hasRest) {
            return new ApiRes<>(200, "非連續時段班無法休息", null);
        }

        if ((req.getRestStart() == null && req.getRestEnd() != null) ||
                (req.getRestStart() != null && req.getRestEnd() == null)) {
            return new ApiRes<>(400, "請輸入完整休息資訊", null);
        }

        if (hasRest && req.getRestStart().isAfter(req.getRestEnd())) {
            return new ApiRes<>(400, "休息結束不可早於開始", null);
        }

        // 計算時數
        final int DAY = 86400;

        int restSec = 0;
        if (hasRest) {
            restSec = (req.getRestEnd().toSecondOfDay() - req.getRestStart().toSecondOfDay() + DAY) % DAY;
        }

        int workSec = (req.getClockOff().toSecondOfDay() - req.getClockOn().toSecondOfDay() + DAY) % DAY;

        double totalHour = Math.max(0, workSec - restSec) / 3600.0;
        totalHour = Math.floor(totalHour * 2.0 + 1e-9) / 2.0;

        if (req.getScore() == null) {
            return new ApiRes<>(400, "score 不可為空", null);
        }

        boolean changed = clockDateDao.reClockPart(
                req.getEmployeeId(), req.getWorkDate(),
                req.getClockOn(), req.getClockOff(),
                req.getRestStart(), req.getRestEnd(),
                totalHour, req.getScore()) == 1;

        if (changed) {
            return new ApiRes<>(200, "補打卡成功", null);
        }
        return new ApiRes<>(500, "未知錯誤", null);
    }
    

}

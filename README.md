package com.example.p01.dao;

import java.time.LocalDate;
import java.time.LocalTime;
import java.util.List;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import com.example.p01.entity.ClockDate;
import com.example.p01.entity.ClockDateId;

import jakarta.transaction.Transactional;

public interface ClockDateDao extends JpaRepository<ClockDate, ClockDateId> {

    // Optional<xxx>: 保證不會有 NPE，但要手動處理 isEmpty
    @Query(value = "select * from clock_date where employee_id = ?1 and work_date = ?2 "//
            + " order by clock_on ", nativeQuery = true)
    public List<ClockDate> getSingleClock(String employeeId, LocalDate workDate);

    @Query(value = "select * from clock_date", nativeQuery = true)
    public List<ClockDate> getAllClock();

    @Query(value = "select * from clock_date where employee_id = ?1", nativeQuery = true)
    public List<ClockDate> getSingleHistoryClock(String employeeId);

    @Query(value = """
            select ps.shift_work_id
            from p01.pre_schedule ps
            join p01.shift_work  sw on sw.shift_work_id = ps.shift_work_id
            where ps.employee_id = :employeeId
              and ps.apply_date  = :workDate
              and ps.is_accept   = 1
              and timestamp(ps.apply_date, :clockOn) >=
                    timestamp(ps.apply_date, sw.start_time) - interval 30 minute
              and timestamp(ps.apply_date, :clockOn) <
                    case when sw.end_time > sw.start_time
                           then timestamp(ps.apply_date, sw.end_time)
                         else timestamp(date_add(ps.apply_date, interval 1 day), sw.end_time)
                    end
            order by ps.shift_work_id
            limit 1
            """, nativeQuery = true)
    Integer findShiftClass(@Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

    @Query(value = """
            select exists (
              /* 命中任一已核可班別的容許時窗（含提前 30 分；支援跨日） */
              select 1
              from p01.pre_schedule ps
              join p01.shift_work  sw on sw.shift_work_id = ps.shift_work_id
              where ps.employee_id   = :employeeId
                and ps.apply_date    = :workDate
                and ps.is_accept     = 1
                and ps.shift_work_id > 0
                and timestamp(ps.apply_date, :clockOn) >=
                      timestamp(ps.apply_date, sw.start_time) - interval 30 minute
                and timestamp(ps.apply_date, :clockOn) <
                      case
                        when sw.end_time > sw.start_time
                          then timestamp(ps.apply_date, sw.end_time)
                        else
                          timestamp(date_add(ps.apply_date, interval 1 day), sw.end_time)
                      end

              union all

              /* 若是連續班（差 1），命中其中任何一班也算有排班 */
              select 1
              from p01.pre_schedule a
              join p01.pre_schedule b
                on a.employee_id = b.employee_id
               and a.apply_date  = b.apply_date
               and a.is_accept   = 1
               and b.is_accept   = 1
               and abs(a.shift_work_id - b.shift_work_id) = 1
              join p01.shift_work sw1 on sw1.shift_work_id = a.shift_work_id
              join p01.shift_work sw2 on sw2.shift_work_id = b.shift_work_id
              where a.employee_id = :employeeId
                and a.apply_date  = :workDate
                and (
                  timestamp(a.apply_date, :clockOn) >=
                    timestamp(a.apply_date, sw1.start_time) - interval 30 minute
                  and timestamp(a.apply_date, :clockOn) <
                    case
                      when sw1.end_time > sw1.start_time
                        then timestamp(a.apply_date, sw1.end_time)
                      else
                        timestamp(date_add(a.apply_date, interval 1 day), sw1.end_time)
                    end
                  or
                  timestamp(b.apply_date, :clockOn) >=
                    timestamp(b.apply_date, sw2.start_time) - interval 30 minute
                  and timestamp(b.apply_date, :clockOn) <
                    case
                      when sw2.end_time > sw2.start_time
                        then timestamp(b.apply_date, sw2.end_time)
                      else
                        timestamp(date_add(b.apply_date, interval 1 day), sw2.end_time)
                    end
                )
              limit 1
            )
            """, nativeQuery = true)
    int checkAcceptPre(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

    @Query(value = """
            select exists (
              /* case 1：同一班別時段內已經有 clock_on */
              select 1
              from p01.pre_schedule ps
              join p01.shift_work sw on sw.shift_work_id = ps.shift_work_id
              join p01.clock_date cd on cd.employee_id = ps.employee_id
                                    and cd.work_date   = ps.apply_date
              where ps.employee_id   = :employeeId
                and ps.apply_date    = :workDate
                and ps.is_accept     = b'1'
                and ps.shift_work_id > 0
                and :clockOn >= subtime(sw.start_time, '00:30:00')
                and :clockOn <  sw.end_time
                and cd.clock_on >= subtime(sw.start_time, '00:30:00')
                and cd.clock_on <  sw.end_time

              union all

              /* case 2：當天存在連續班（差 1），則當天若已有任何 clock_date，就視為已打 */
              select 1
              from p01.clock_date c2
              where c2.employee_id = :employeeId
                and c2.work_date   = :workDate
                and exists (
                  select 1
                  from p01.pre_schedule a
                  join p01.pre_schedule b
                    on a.employee_id = b.employee_id
                   and a.apply_date  = b.apply_date
                   and a.is_accept   = b'1'
                   and b.is_accept   = b'1'
                   and abs(a.shift_work_id - b.shift_work_id) = 1
                  where a.employee_id = :employeeId
                    and a.apply_date  = :workDate
                )
              limit 1
            )
            """, nativeQuery = true)
    int checkExistsOn(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

    @Query(value = """
            select exists (
              select 1
              from p01.pre_schedule ps
              join p01.shift_work sw on sw.shift_work_id = ps.shift_work_id
              join p01.clock_date cd on cd.employee_id = ps.employee_id
                                    and cd.work_date   = ps.apply_date
              where ps.employee_id   = :employeeId
                and ps.apply_date    = :workDate
                and ps.is_accept     = 1
                and ps.shift_work_id > 0
                /* 參數 clockOn 命中此班別的容許時窗（含提前 30 分；跨日） */
                and timestamp(ps.apply_date, :clockOn) >=
                      timestamp(ps.apply_date, sw.start_time) - interval 30 minute
                and timestamp(ps.apply_date, :clockOn) <
                      case when sw.end_time > sw.start_time
                             then timestamp(ps.apply_date, sw.end_time)
                           else timestamp(date_add(ps.apply_date, interval 1 day), sw.end_time)
                      end
                /* 已存在的 clock_on 也命中同一個班別視窗（跨日） */
                and timestamp(ps.apply_date, cd.clock_on) >=
                      timestamp(ps.apply_date, sw.start_time) - interval 30 minute
                and timestamp(ps.apply_date, cd.clock_on) <
                      case when sw.end_time > sw.start_time
                             then timestamp(ps.apply_date, sw.end_time)
                           else timestamp(date_add(ps.apply_date, interval 1 day), sw.end_time)
                      end
              limit 1
            )
            """, nativeQuery = true)
    int checkExistsOn2(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

    @Transactional
    @Modifying
    @Query(value = """
            insert into p01.clock_date (employee_id, work_date, clock_on, has_double)
            select
              :employeeId, :workDate, :clockOn,
              case when exists (
                select 1 from p01.holiday h
                where h.holiday_date = :workDate
                  and h.has_double   = 1
                  and h.has_workday  = 1
              ) then b'1' else b'0' end
            from dual
            where
              (
                /* 連續班（相鄰，差 1）：命中任一班別的容許時窗（含提前 30 分） */
                exists (
                  select 1
                  from p01.pre_schedule a
                  join p01.pre_schedule b
                    on a.employee_id = b.employee_id
                   and a.apply_date  = b.apply_date
                   and a.is_accept   = 1
                   and b.is_accept   = 1
                   and abs(a.shift_work_id - b.shift_work_id) = 1
                  join p01.shift_work sw1 on sw1.shift_work_id = a.shift_work_id
                  join p01.shift_work sw2 on sw2.shift_work_id = b.shift_work_id
                  where a.employee_id = :employeeId
                    and a.apply_date  = :workDate
                    and (
                      (
                        timestamp(:workDate, :clockOn) >=
                          timestamp(:workDate, sw1.start_time) - interval 30 minute
                        and timestamp(:workDate, :clockOn) <
                          case
                            when sw1.end_time > sw1.start_time
                              then timestamp(:workDate, sw1.end_time)
                            else
                              timestamp(date_add(:workDate, interval 1 day), sw1.end_time)
                          end
                      )
                      or
                      (
                        timestamp(:workDate, :clockOn) >=
                          timestamp(:workDate, sw2.start_time) - interval 30 minute
                        and timestamp(:workDate, :clockOn) <
                          case
                            when sw2.end_time > sw2.start_time
                              then timestamp(:workDate, sw2.end_time)
                            else
                              timestamp(date_add(:workDate, interval 1 day), sw2.end_time)
                          end
                      )
                    )
                )
                or
                /* 非連續班：命中該班別的容許時窗（含提前 30 分） */
                exists (
                  select 1
                  from p01.pre_schedule ps
                  join p01.shift_work  sw on sw.shift_work_id = ps.shift_work_id
                  where ps.employee_id   = :employeeId
                    and ps.apply_date    = :workDate
                    and ps.is_accept     = 1
                    and ps.shift_work_id > 0
                    and timestamp(:workDate, :clockOn) >=
                          timestamp(:workDate, sw.start_time) - interval 30 minute
                    and timestamp(:workDate, :clockOn) <
                          case
                            when sw.end_time > sw.start_time
                              then timestamp(:workDate, sw.end_time)
                            else
                              timestamp(date_add(:workDate, interval 1 day), sw.end_time)
                          end
                )
              )
              /* 防止同一時刻重複打卡（同員工同日同時間） */
              and not exists (
                select 1 from p01.clock_date cd
                where cd.employee_id = :employeeId
                  and cd.work_date   = :workDate
                  and cd.clock_on    = :clockOn
              )
            limit 1
            """, nativeQuery = true)
    int addClockOnWhenNotContinuous(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

    @Query(value = """
            select exists (
            select 1
            from p01.pre_schedule as a
            join p01.pre_schedule as b
            on a.employee_id = b.employee_id
            and a.apply_date = b.apply_date
            and a.is_accept = b'1'
            and b.is_accept = b'1'
            and abs(a.shift_work_id - b.shift_work_id) = 1
            where a.employee_id = :employeeId
            and a.apply_date = :workDate )
            """, nativeQuery = true)
    public int hasContinuousShift(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate);

    @Transactional
    @Modifying
    @Query(value = """
            update p01.clock_date as cd
            set cd.rest_start = :restStart
            where cd.employee_id = :employeeId
            and cd.work_date = :workDate
            and cd.rest_start is null
            order by cd.clock_on desc
            limit 1
            """, nativeQuery = true)
    public int addRestStart(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restStart") LocalTime restStart);

    @Transactional
    @Modifying
    @Query(value = """
            update p01.clock_date as cd
            set cd.rest_end = :restEnd
            where cd.employee_id = :employeeId
            and cd.work_date = :workDate
            and cd.rest_start is not null
            and cd.rest_end is null
            order by cd.clock_on desc
            limit 1
            """, nativeQuery = true)
    public int addRestEnd(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restEnd") LocalTime restEnd);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId
            and work_date = :workDate
            and rest_start is not null
            and rest_end is null)
            """, nativeQuery = true)
    public int existsRestStart(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restStart") LocalTime restStart);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId
            and work_date = :workDate
            and rest_end is not null)
            """, nativeQuery = true)
    public int existsRestEnd(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restEnd") LocalTime restEnd);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId
            and work_date = :workDate
            and clock_on is not null)
            """, nativeQuery = true)
    public int hasClockOn(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId
            and work_date = :workDate
            and clock_off is not null)
            """, nativeQuery = true)
    public int hasClockOff(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId
            and work_date = :workDate
            and clock_on is not null
            and rest_start is not null)
            """, nativeQuery = true)
    public int hasRestStart(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate);

    @Query(value = """
            select exists (
            select 1
              from p01.pre_schedule a
              join p01.pre_schedule b
                on a.employee_id = b.employee_id
               and a.apply_date  = b.apply_date
               and a.is_accept   = 1
               and b.is_accept   = 1
               and abs(a.shift_work_id - b.shift_work_id) = 1
              join p01.shift_work sw1 on sw1.shift_work_id = a.shift_work_id
              join p01.shift_work sw2 on sw2.shift_work_id = b.shift_work_id
              where a.employee_id = :employeeId
                and a.apply_date  = :workDate
                and (
                  ( timestamp(:workDate, :restStart) >= timestamp(:workDate, sw1.start_time) - interval 30 minute
                    and timestamp(:workDate, :restStart) <
                        case when sw1.end_time > sw1.start_time
                               then timestamp(:workDate, sw1.end_time)
                             else timestamp(date_add(:workDate, interval 1 day), sw1.end_time)
                        end )
                  or
                  ( timestamp(:workDate, :restStart) >= timestamp(:workDate, sw2.start_time) - interval 30 minute
                    and timestamp(:workDate, :restStart) <
                        case when sw2.end_time > sw2.start_time
                               then timestamp(:workDate, sw2.end_time)
                             else timestamp(date_add(:workDate, interval 1 day), sw2.end_time)
                        end )
                )
              limit 1
            )
            """, nativeQuery = true)
    int canStartRest(@Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restStart") LocalTime restStart);

    @Query(value = """
            select exists (
              select 1
              from p01.pre_schedule a
              join p01.pre_schedule b
                on a.employee_id = b.employee_id
               and a.apply_date  = b.apply_date
               and a.is_accept   = 1
               and b.is_accept   = 1
               and abs(a.shift_work_id - b.shift_work_id) = 1
              join p01.shift_work sw1 on sw1.shift_work_id = a.shift_work_id
              join p01.shift_work sw2 on sw2.shift_work_id = b.shift_work_id
              join p01.clock_date cd
                on cd.employee_id = a.employee_id
               and cd.work_date   = a.apply_date
               and cd.rest_start is not null
               and cd.rest_end   is null
              where a.employee_id = :employeeId
                and a.apply_date  = :workDate
                and (
                  ( timestamp(:workDate, :restEnd) >= timestamp(:workDate, sw1.start_time) - interval 30 minute
                    and timestamp(:workDate, :restEnd) <
                        case when sw1.end_time > sw1.start_time
                               then timestamp(:workDate, sw1.end_time)
                             else timestamp(date_add(:workDate, interval 1 day), sw1.end_time)
                        end )
                  or
                  ( timestamp(:workDate, :restEnd) >= timestamp(:workDate, sw2.start_time) - interval 30 minute
                    and timestamp(:workDate, :restEnd) <
                        case when sw2.end_time > sw2.start_time
                               then timestamp(:workDate, sw2.end_time)
                             else timestamp(date_add(:workDate, interval 1 day), sw2.end_time)
                        end )
                )
                and (
                  /* 把 rest_end 轉成正確的日期時間後再跟 rest_start 比（支援跨日） */
                  case when :restEnd >= cd.rest_start
                         then timestamp(:workDate, :restEnd)
                       else timestamp(date_add(:workDate, interval 1 day), :restEnd)
                  end
                ) >= timestamp(:workDate, cd.rest_start)
              limit 1
            )
            """, nativeQuery = true)
    int canEndRest(@Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("restEnd") LocalTime restEnd);

    @Query(value = """
            select exists (
              /* 用 atTime = COALESCE(:clockOn, :clockOff) 做檢查 */
              select 1
              from p01.pre_schedule ps
              join p01.shift_work sw on sw.shift_work_id = ps.shift_work_id
              where ps.employee_id   = :employeeId
                and ps.apply_date    = :workDate
                and ps.is_accept     = 1
                and ps.shift_work_id > 0
                and timestamp(ps.apply_date, COALESCE(:clockOn, :clockOff)) >=
                      timestamp(ps.apply_date, sw.start_time) - interval 30 minute
                and timestamp(ps.apply_date, COALESCE(:clockOn, :clockOff)) <
                      case when sw.end_time > sw.start_time
                             then timestamp(ps.apply_date, sw.end_time)
                           else timestamp(date_add(ps.apply_date, interval 1 day), sw.end_time)
                      end
              union all
              /* 連續班分支同樣把 :clockOn 換成 COALESCE(:clockOn, :clockOff) */
              select 1
              from p01.pre_schedule a
              join p01.pre_schedule b
                on a.employee_id = b.employee_id
               and a.apply_date  = b.apply_date
               and a.is_accept   = 1
               and b.is_accept   = 1
               and abs(a.shift_work_id - b.shift_work_id) = 1
              join p01.shift_work sw1 on sw1.shift_work_id = a.shift_work_id
              join p01.shift_work sw2 on sw2.shift_work_id = b.shift_work_id
              where a.employee_id = :employeeId
                and a.apply_date  = :workDate
                and (
                  timestamp(a.apply_date, COALESCE(:clockOn, :clockOff)) >=
                    timestamp(a.apply_date, sw1.start_time) - interval 30 minute
                  and timestamp(a.apply_date, COALESCE(:clockOn, :clockOff)) <
                    case when sw1.end_time > sw1.start_time
                           then timestamp(a.apply_date, sw1.end_time)
                         else timestamp(date_add(a.apply_date, interval 1 day), sw1.end_time)
                    end
                  or
                  timestamp(b.apply_date, COALESCE(:clockOn, :clockOff)) >=
                    timestamp(b.apply_date, sw2.start_time) - interval 30 minute
                  and timestamp(b.apply_date, COALESCE(:clockOn, :clockOff)) <
                    case when sw2.end_time > sw2.start_time
                           then timestamp(b.apply_date, sw2.end_time)
                         else timestamp(date_add(b.apply_date, interval 1 day), sw2.end_time)
                    end
                )
              limit 1
            )
            """, nativeQuery = true)
    int checkAcceptPreEither(@Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn,
            @Param("clockOff") LocalTime clockOff);

    @Query(value = """
            select exists (
            select 1 from p01.clock_date
            where employee_id = :employeeId and work_date = :workDate)
            """, nativeQuery = true)
    public int checkExistsClockOn(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate);

    @Query(value = """
            update p01.clock_date as cd
            set cd.clock_off = :clockOff
            where cd.employee_id = :employeeId
            and cd.clock_date = :clockDate
            and cd.clock_on is not null
            and cd.clock_off is null
            """, nativeQuery = true)
    public int reClockOff(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOff") LocalTime clockOff);

    @Modifying
    @Transactional
    @Query(value = """
            insert into p01.clock_date
            (employee_id, work_date, clock_on,
             clock_off, rest_start, rest_end,
             total_hour, has_double, score)
            select
            :employeeId, :workDate, :clockOn, :clockOff,
            :restStart, :restEnd, :totalHour,
            case when exists (
            select 1 from p01.holiday as h
            where h.holiday_date = :workDate
            and h.has_double = 1
            and h.has_workday = 1)
            then b'1' else b'0' end,
            :score
            """, nativeQuery = true)
    public int reClockAll(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn,
            @Param("clockOff") LocalTime clockOff,
            @Param("restStart") LocalTime restStart,
            @Param("restEnd") LocalTime restEnd,
            @Param("totalHour") Double totalHour,
            @Param("score") Integer score);

    @Modifying
    @Transactional
    @Query(value = """
            update p01.clock_date
            set clock_on = :clockOn,
                clock_off = :clockOff,
                rest_start = :restStart,
                rest_end = :restEnd,
                total_hour = :totalHour,
                score = :score
            where employee_id = :employeeId
                and work_date = :workDate
                and clock_on = :clockOn
            order by clock_on desc
            limit 1
            """, nativeQuery = true)
    public int reClockPart(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn,
            @Param("clockOff") LocalTime clockOff,
            @Param("restStart") LocalTime restStart,
            @Param("restEnd") LocalTime restEnd,
            @Param("totalHour") Double totalHour,
            @Param("score") Integer score);

    @Query(value = """
            select case when exists (
              select 1
              from p01.clock_date
              where employee_id = :employeeId
                and work_date   = :workDate
                and clock_on = :clockOn
              limit 1
            ) then 1 else 0 end
            """, nativeQuery = true)
    public int checkReClockExists(
            @Param("employeeId") String employeeId,
            @Param("workDate") LocalDate workDate,
            @Param("clockOn") LocalTime clockOn);

}

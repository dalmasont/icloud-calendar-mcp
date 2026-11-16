import { describe, expect, test } from "bun:test";
import {
  excludeOccurrence,
  parseEvents,
  RecurrenceBudget,
} from "../src/ical.ts";

const NOW = new Date("2026-11-16T12:00:00Z");

const calendar = (...lines: string[]) =>
  [
    "BEGIN:VCALENDAR",
    "VERSION:2.0",
    "PRODID:-//Apple Inc.//iCloud//EN",
    ...lines,
    "END:VCALENDAR",
  ].join("\r\n");

const lines = (ics: string) => ics.split("\r\n");

/** iCloud stores IANA TZIDs and may omit VTIMEZONE, so none is included here */
const DAILY_SAO_PAULO = calendar(
  "BEGIN:VEVENT",
  "UID:agenda",
  "DTSTAMP:20261001T000000Z",
  "SEQUENCE:3",
  "DTSTART;TZID=America/Sao_Paulo:20261116T080000",
  "DTEND;TZID=America/Sao_Paulo:20261116T230000",
  "RRULE:FREQ=DAILY",
  "SUMMARY:Agenda Aberta",
  "BEGIN:VALARM",
  "ACTION:DISPLAY",
  "DESCRIPTION:Reminder",
  "TRIGGER:-PT15M",
  "END:VALARM",
  "END:VEVENT",
);

const WEEK = {
  start: new Date("2026-11-15T00:00:00-03:00"),
  end: new Date("2026-11-21T00:00:00-03:00"),
  localOffsetSeconds: -3 * 60 * 60,
};

const startsIn = (ics: string, period = WEEK) =>
  parseEvents(ics, period, new RecurrenceBudget()).map((e) => e.start);

describe("excludeOccurrence: timed series with a time zone", () => {
  test("adds an EXDATE with the series' TZID and keeps the rest of the series", () => {
    const { ics, changed } = excludeOccurrence(
      DAILY_SAO_PAULO,
      "2026-11-17T08:00:00-03:00",
      NOW,
    );

    expect(changed).toBe(true);
    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261117T080000",
    );
    expect(lines(ics)).toContain("RRULE:FREQ=DAILY");
    expect(lines(ics)).toContain(
      "DTSTART;TZID=America/Sao_Paulo:20261116T080000",
    );
    expect(lines(ics)).toContain("TRIGGER:-PT15M");
    expect(lines(ics)).toContain("SEQUENCE:4");
    expect(lines(ics)).toContain("DTSTAMP:20261116T120000Z");
  });

  test("removes only that occurrence when the series is expanded", () => {
    const { ics } = excludeOccurrence(
      DAILY_SAO_PAULO,
      "2026-11-17T08:00:00-03:00",
      NOW,
    );

    expect(startsIn(ics)).toEqual([
      "2026-11-16T08:00:00-03:00",
      "2026-11-18T08:00:00-03:00",
      "2026-11-19T08:00:00-03:00",
      "2026-11-20T08:00:00-03:00",
    ]);
  });

  test("accepts the same instant written in another offset", () => {
    const { ics } = excludeOccurrence(
      DAILY_SAO_PAULO,
      "2026-11-17T11:00:00Z",
      NOW,
    );

    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261117T080000",
    );
  });

  test("a date alone picks the occurrence that starts that day in the series' zone", () => {
    const byDate = excludeOccurrence(DAILY_SAO_PAULO, "2026-11-17", NOW);
    const byInstant = excludeOccurrence(
      DAILY_SAO_PAULO,
      "2026-11-17T08:00:00-03:00",
      NOW,
    );

    expect(byDate.ics).toBe(byInstant.ics);
  });

  test("can exclude the first occurrence without moving DTSTART", () => {
    const { ics } = excludeOccurrence(DAILY_SAO_PAULO, "2026-11-16", NOW);

    expect(lines(ics)).toContain(
      "DTSTART;TZID=America/Sao_Paulo:20261116T080000",
    );
    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261116T080000",
    );
    expect(startsIn(ics)[0]).toBe("2026-11-17T08:00:00-03:00");
  });

  test("uses the offset in force on that day across a DST change", () => {
    const weeklyNewYork = calendar(
      "BEGIN:VEVENT",
      "UID:ny",
      "DTSTAMP:20261001T000000Z",
      "DTSTART;TZID=America/New_York:20261026T090000",
      "DTEND;TZID=America/New_York:20261026T100000",
      "RRULE:FREQ=WEEKLY",
      "SUMMARY:Standup",
      "END:VEVENT",
    );

    const { ics } = excludeOccurrence(
      weeklyNewYork,
      "2026-11-02T09:00:00-05:00",
      NOW,
    );

    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/New_York:20261102T090000",
    );
    // 09:00-04:00 is 08:00 in New York after the clocks went back: no such slot
    expect(() =>
      excludeOccurrence(weeklyNewYork, "2026-11-02T09:00:00-04:00", NOW),
    ).toThrow("Unknown occurrence");
  });

  test("works when the stored data carries its own VTIMEZONE", () => {
    const withZone = calendar(
      "BEGIN:VTIMEZONE",
      "TZID:America/Sao_Paulo",
      "BEGIN:STANDARD",
      "DTSTART:19700101T000000",
      "TZOFFSETFROM:-0300",
      "TZOFFSETTO:-0300",
      "TZNAME:-03",
      "END:STANDARD",
      "END:VTIMEZONE",
      "BEGIN:VEVENT",
      "UID:agenda",
      "DTSTAMP:20261001T000000Z",
      "DTSTART;TZID=America/Sao_Paulo:20261116T080000",
      "DTEND;TZID=America/Sao_Paulo:20261116T230000",
      "RRULE:FREQ=DAILY",
      "SUMMARY:Agenda Aberta",
      "END:VEVENT",
    );

    const { ics } = excludeOccurrence(withZone, "2026-11-17", NOW);

    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261117T080000",
    );
    expect(lines(ics)).toContain("TZID:America/Sao_Paulo");
  });
});

describe("excludeOccurrence: other series forms", () => {
  test("UTC series get a UTC EXDATE", () => {
    const utc = calendar(
      "BEGIN:VEVENT",
      "UID:u",
      "DTSTAMP:20261001T000000Z",
      "DTSTART:20261116T110000Z",
      "DTEND:20261116T120000Z",
      "RRULE:FREQ=DAILY",
      "SUMMARY:x",
      "END:VEVENT",
    );

    const { ics } = excludeOccurrence(utc, "2026-11-17T11:00:00Z", NOW);

    expect(lines(ics)).toContain("EXDATE:20261117T110000Z");
  });

  test("all-day series get a DATE EXDATE and only match dates", () => {
    const allDay = calendar(
      "BEGIN:VEVENT",
      "UID:d",
      "DTSTAMP:20261001T000000Z",
      "DTSTART;VALUE=DATE:20261116",
      "DTEND;VALUE=DATE:20261117",
      "RRULE:FREQ=DAILY",
      "SUMMARY:x",
      "END:VEVENT",
    );

    const { ics } = excludeOccurrence(allDay, "2026-11-17", NOW);

    expect(lines(ics)).toContain("EXDATE;VALUE=DATE:20261117");
    expect(startsIn(ics)).not.toContain("2026-11-17");
    expect(() =>
      excludeOccurrence(allDay, "2026-11-17T00:00:00-03:00", NOW),
    ).toThrow("Unknown occurrence");
  });

  test("floating series get a floating EXDATE and match offset-less times", () => {
    const floating = calendar(
      "BEGIN:VEVENT",
      "UID:f",
      "DTSTAMP:20261001T000000Z",
      "DTSTART:20261116T080000",
      "DTEND:20261116T090000",
      "RRULE:FREQ=DAILY",
      "SUMMARY:x",
      "END:VEVENT",
    );

    const { ics } = excludeOccurrence(floating, "2026-11-17T08:00:00", NOW);

    expect(lines(ics)).toContain("EXDATE:20261117T080000");
    expect(() =>
      excludeOccurrence(floating, "2026-11-17T08:00:00-03:00", NOW),
    ).toThrow("Unknown occurrence");
  });
});

describe("excludeOccurrence: occurrences that were already changed", () => {
  const WITH_OVERRIDE = DAILY_SAO_PAULO.replace(
    "END:VCALENDAR",
    [
      "BEGIN:VEVENT",
      "UID:agenda",
      "DTSTAMP:20261001T000000Z",
      "RECURRENCE-ID;TZID=America/Sao_Paulo:20261118T080000",
      "DTSTART;TZID=America/Sao_Paulo:20261118T100000",
      "DTEND;TZID=America/Sao_Paulo:20261118T230000",
      "SUMMARY:Agenda Aberta (late)",
      "END:VEVENT",
      "END:VCALENDAR",
    ].join("\r\n"),
  );

  test("the moved occurrence can be excluded by the start list_events shows", () => {
    expect(startsIn(WITH_OVERRIDE)).toContain("2026-11-18T10:00:00-03:00");

    const { ics } = excludeOccurrence(
      WITH_OVERRIDE,
      "2026-11-18T10:00:00-03:00",
      NOW,
    );

    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261118T080000",
    );
    expect(lines(ics).some((l) => l.startsWith("RECURRENCE-ID"))).toBe(false);
    expect(startsIn(ics).filter((s) => s.startsWith("2026-11-18"))).toEqual([]);
    expect(startsIn(ics)).toContain("2026-11-19T08:00:00-03:00");
  });

  test("a date alone finds the moved occurrence without counting it twice", () => {
    const { ics } = excludeOccurrence(WITH_OVERRIDE, "2026-11-18", NOW);

    expect(lines(ics).filter((l) => l.startsWith("EXDATE"))).toEqual([
      "EXDATE;TZID=America/Sao_Paulo:20261118T080000",
    ]);
    expect(lines(ics).some((l) => l.startsWith("RECURRENCE-ID"))).toBe(false);
  });

  test("other overrides stay untouched", () => {
    const { ics } = excludeOccurrence(WITH_OVERRIDE, "2026-11-17", NOW);

    expect(lines(ics)).toContain(
      "RECURRENCE-ID;TZID=America/Sao_Paulo:20261118T080000",
    );
    expect(startsIn(ics)).toContain("2026-11-18T10:00:00-03:00");
  });
});

describe("excludeOccurrence: repeating and invalid requests", () => {
  test("excluding an occurrence twice changes nothing the second time", () => {
    const first = excludeOccurrence(DAILY_SAO_PAULO, "2026-11-17", NOW);
    const second = excludeOccurrence(first.ics, "2026-11-17", NOW);

    expect(second.changed).toBe(false);
    expect(second.ics).toBe(first.ics);
  });

  test("excluding several days keeps one EXDATE per day", () => {
    const first = excludeOccurrence(DAILY_SAO_PAULO, "2026-11-17", NOW);
    const second = excludeOccurrence(first.ics, "2026-11-18", NOW);

    expect(lines(second.ics).filter((l) => l.startsWith("EXDATE"))).toEqual([
      "EXDATE;TZID=America/Sao_Paulo:20261117T080000",
      "EXDATE;TZID=America/Sao_Paulo:20261118T080000",
    ]);
    expect(lines(second.ics)).toContain("SEQUENCE:5");
  });

  test("refuses an event that does not repeat", () => {
    const single = calendar(
      "BEGIN:VEVENT",
      "UID:s",
      "DTSTAMP:20261001T000000Z",
      "DTSTART:20261116T110000Z",
      "DTEND:20261116T120000Z",
      "SUMMARY:x",
      "END:VEVENT",
    );

    expect(() =>
      excludeOccurrence(single, "2026-11-16T11:00:00Z", NOW),
    ).toThrow("not recurring");
  });

  test("refuses a day the series does not have", () => {
    const threeDays = DAILY_SAO_PAULO.replace(
      "RRULE:FREQ=DAILY",
      "RRULE:FREQ=DAILY;COUNT=3",
    );

    expect(() => excludeOccurrence(threeDays, "2026-11-20", NOW)).toThrow(
      "Unknown occurrence",
    );
    expect(() => excludeOccurrence(threeDays, "2026-11-10", NOW)).toThrow(
      "Unknown occurrence",
    );
  });

  test("refuses a time that is not one of the series' slots", () => {
    expect(() =>
      excludeOccurrence(DAILY_SAO_PAULO, "2026-11-17T09:00:00-03:00", NOW),
    ).toThrow("Unknown occurrence");
  });

  test("asks for the full start when a date matches several occurrences", () => {
    const hourly = calendar(
      "BEGIN:VEVENT",
      "UID:h",
      "DTSTAMP:20261001T000000Z",
      "DTSTART;TZID=America/Sao_Paulo:20261116T080000",
      "DTEND;TZID=America/Sao_Paulo:20261116T083000",
      "RRULE:FREQ=HOURLY;COUNT=5",
      "SUMMARY:x",
      "END:VEVENT",
    );

    expect(() => excludeOccurrence(hourly, "2026-11-16", NOW)).toThrow(
      "several occurrences",
    );
    const { ics } = excludeOccurrence(hourly, "2026-11-16T09:00:00-03:00", NOW);
    expect(lines(ics)).toContain(
      "EXDATE;TZID=America/Sao_Paulo:20261116T090000",
    );
  });

  test("gives up on a date too far from the series start instead of looping", () => {
    expect(() => excludeOccurrence(DAILY_SAO_PAULO, "2200-01-01", NOW)).toThrow(
      "too far",
    );
  });

  test.each([
    "not a date",
    "2026-13-40",
    "2026-11-17T08:00:00+0300",
    "2026-11-17T25:00:00Z",
    "",
  ])("rejects an invalid occurrence: %p", (value) => {
    expect(() => excludeOccurrence(DAILY_SAO_PAULO, value, NOW)).toThrow(
      "Invalid occurrence",
    );
  });
});

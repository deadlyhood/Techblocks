/*
  eco_tracker.c
  Simple terminal-based environmental impact tracker for beginners
  - Uses standard C library only
  - Stores daily entries in eco_log.csv (created in the same directory)
  - Shows daily/weekly/monthly footprints, habit tracking, simple ASCII charts,
    color-coded status and action suggestions.

  Compile:
    gcc eco_tracker.c -o eco_tracker

  Run:
    ./eco_tracker

  Author: ChatGPT (Acting as a professional C developer)
  Notes: All numeric "emission factors" are simple illustrative approximations,
         intended to teach and motivate — not for high-precision accounting.
*/

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <ctype.h>

/* ---------------------------
   Configuration / constants
   --------------------------- */
/* Output file for logs */
#define LOG_FILE "eco_log.csv"

/* Emission factors (kg CO2 equivalents) -- simple illustrative approximations */
#define EF_CAR_PER_KM      0.21   /* car (kg CO2 per km) */
#define EF_BUS_PER_KM      0.05   /* bus (kg CO2 per km) */
#define EF_TRAIN_PER_KM    0.04   /* train (kg CO2 per km) */
#define EF_FLIGHT_PER_KM   0.15   /* flight (kg CO2 per km) */
#define EF_ELECTRICITY_KWH 0.85   /* electricity (kg CO2 per kWh) */
#define EF_PLASTIC_PER_KG  6.00   /* plastic production & waste (kg CO2 per kg) */
#define EF_MEAT_MEAL       5.0    /* meat-heavy meal (kg CO2 per meal) */
#define EF_VEG_MEAL        1.5    /* vegetarian meal (kg CO2 per meal) */
#define EF_FISH_MEAL       3.0    /* fish meal (kg CO2 per meal) */

/* Visual thresholds (kg/day) */
#define LOW_THRESHOLD    10.0
#define MEDIUM_THRESHOLD 30.0
#define MAX_BAR_WIDTH    40     /* for ASCII bar display */

/* ANSI color codes (may work in most terminals) */
#define COLOR_RESET  "\x1b[0m"
#define COLOR_GREEN  "\x1b[32m"
#define COLOR_YELLOW "\x1b[33m"
#define COLOR_RED    "\x1b[31m"
#define COLOR_CYAN   "\x1b[36m"

/* ---------------------------
   Data structures
   --------------------------- */

typedef struct {
    char date[11]; /* "YYYY-MM-DD" + null */
    double car_km;
    double bus_km;
    double train_km;
    double flight_km;
    double electricity_kwh;
    double plastic_kg;
    int meat_meals;
    int veg_meals;
    int fish_meals;
    int recycled;           /* 1 = yes, 0 = no */
    int used_public_transport_count; /* count of public transport actions that day */
    int saved_electricity_actions;   /* simple count of user-claimed electricity-saving actions */
    double footprint_kg;    /* total computed footprint for the entry */
} DayEntry;

/* ---------------------------
   Utility functions
   --------------------------- */

/* Get today's date as "YYYY-MM-DD" */
void get_today_str(char *buf, size_t bufsize) {
    time_t t = time(NULL);
    struct tm tm = *localtime(&t);
    snprintf(buf, bufsize, "%04d-%02d-%02d", tm.tm_year+1900, tm.tm_mon+1, tm.tm_mday);
}

/* Parse a date "YYYY-MM-DD" into a time_t at midnight local time. Returns -1 on failure. */
time_t parse_date_midnight(const char *datestr) {
    struct tm tm;
    memset(&tm, 0, sizeof(tm));
    if (sscanf(datestr, "%d-%d-%d", &tm.tm_year, &tm.tm_mon, &tm.tm_mday) != 3) {
        return (time_t)-1;
    }
    tm.tm_year -= 1900;
    tm.tm_mon -= 1;
    tm.tm_hour = 0;
    tm.tm_min = 0;
    tm.tm_sec = 0;
    tm.tm_isdst = -1;
    return mktime(&tm); /* returns -1 on failure */
}

/* Safe input line (trim newline) */
void input_line(char *buf, size_t size) {
    if (fgets(buf, size, stdin) == NULL) {
        buf[0] = '\0';
        return;
    }
    size_t len = strlen(buf);
    if (len && buf[len-1] == '\n') buf[len-1] = '\0';
}

/* Prompt for a double with default value and simple validation */
double prompt_double(const char *prompt, double default_val) {
    char line[128];
    double val;
    while (1) {
        if (default_val >= 0.0) {
            printf("%s (default %.2f): ", prompt, default_val);
        } else {
            printf("%s: ", prompt);
        }
        input_line(line, sizeof(line));
        if (strlen(line) == 0) {
            if (default_val >= 0.0) return default_val;
            else { printf("Please enter a value.\n"); continue; }
        }
        char *endptr;
        val = strtod(line, &endptr);
        if (endptr == line) {
            printf("Invalid number. Try again.\n");
            continue;
        }
        if (val < 0) { printf("Please enter a non-negative number.\n"); continue; }
        return val;
    }
}

/* Prompt for an integer with default */
int prompt_int(const char *prompt, int default_val) {
    char line[128];
    int val;
    while (1) {
        if (default_val >= 0) {
            printf("%s (default %d): ", prompt, default_val);
        } else {
            printf("%s: ", prompt);
        }
        input_line(line, sizeof(line));
        if (strlen(line) == 0) {
            if (default_val >= 0) return default_val;
            else { printf("Please enter a value.\n"); continue; }
        }
        char *endptr;
        val = (int)strtol(line, &endptr, 10);
        if (endptr == line) { printf("Invalid integer. Try again.\n"); continue; }
        if (val < 0) { printf("Please enter a non-negative integer.\n"); continue; }
        return val;
    }
}

/* Prompt for yes/no */
int prompt_yesno(const char *prompt, int default_yes) {
    char line[16];
    while (1) {
        if (default_yes) printf("%s (Y/n): ", prompt);
        else printf("%s (y/N): ", prompt);
        input_line(line, sizeof(line));
        if (strlen(line) == 0) return default_yes;
        if (tolower(line[0]) == 'y') return 1;
        if (tolower(line[0]) == 'n') return 0;
        printf("Please answer y or n.\n");
    }
}

/* ---------------------------
   Core calculation functions
   --------------------------- */

/* Compute footprint components and total for a DayEntry */
double calculate_footprint(DayEntry *d) {
    double car = d->car_km * EF_CAR_PER_KM;
    double bus = d->bus_km * EF_BUS_PER_KM;
    double train = d->train_km * EF_TRAIN_PER_KM;
    double flight = d->flight_km * EF_FLIGHT_PER_KM;
    double elec = d->electricity_kwh * EF_ELECTRICITY_KWH;
    double plastic = d->plastic_kg * EF_PLASTIC_PER_KG;
    double meals = d->meat_meals * EF_MEAT_MEAL + d->veg_meals * EF_VEG_MEAL + d->fish_meals * EF_FISH_MEAL;

    double total = car + bus + train + flight + elec + plastic + meals;
    d->footprint_kg = total;
    return total;
}

/* Determine top contributor (returns string constant) */
const char* top_contributor(DayEntry *d) {
    double car = d->car_km * EF_CAR_PER_KM;
    double transit = d->bus_km * EF_BUS_PER_KM + d->train_km * EF_TRAIN_PER_KM;
    double flight = d->flight_km * EF_FLIGHT_PER_KM;
    double electricity = d->electricity_kwh * EF_ELECTRICITY_KWH;
    double plastic = d->plastic_kg * EF_PLASTIC_PER_KG;
    double food = d->meat_meals * EF_MEAT_MEAL + d->veg_meals * EF_VEG_MEAL + d->fish_meals * EF_FISH_MEAL;

    double values[6] = {car, transit, flight, electricity, plastic, food};
    const char *names[6] = {"Car travel", "Public transit", "Flights", "Electricity", "Plastic", "Food (meals)"};
    int idx = 0;
    for (int i = 1; i < 6; ++i) {
        if (values[i] > values[idx]) idx = i;
    }
    return names[idx];
}

/* ---------------------------
   File handling: CSV storage
   Format header line:
   date,car_km,bus_km,train_km,flight_km,electricity_kwh,plastic_kg,meat_meals,veg_meals,fish_meals,recycled,public_transport_count,saved_electricity_actions,footprint_kg
   --------------------------- */

void ensure_log_header() {
    FILE *f = fopen(LOG_FILE, "r");
    if (f == NULL) {
        /* create and write header */
        f = fopen(LOG_FILE, "w");
        if (!f) {
            fprintf(stderr, "Error: cannot create log file %s\n", LOG_FILE);
            return;
        }
        fprintf(f, "date,car_km,bus_km,train_km,flight_km,electricity_kwh,plastic_kg,meat_meals,veg_meals,fish_meals,recycled,public_transport_count,saved_electricity_actions,footprint_kg\n");
        fclose(f);
    } else {
        fclose(f);
    }
}

/* Append a DayEntry to CSV log */
int append_entry_to_log(DayEntry *d) {
    ensure_log_header();
    FILE *f = fopen(LOG_FILE, "a");
    if (!f) {
        fprintf(stderr, "Error: cannot write to log file.\n");
        return 0;
    }
    fprintf(f, "%s,%.2f,%.2f,%.2f,%.2f,%.2f,%.2f,%d,%d,%d,%d,%d,%d,%.2f\n",
        d->date,
        d->car_km, d->bus_km, d->train_km, d->flight_km,
        d->electricity_kwh, d->plastic_kg,
        d->meat_meals, d->veg_meals, d->fish_meals,
        d->recycled, d->used_public_transport_count, d->saved_electricity_actions,
        d->footprint_kg
    );
    fclose(f);
    return 1;
}

/* Read all entries from log into dynamic array. Returns number read; allocate arr with malloc and set *out_arr. Caller frees. */
int read_all_entries(DayEntry **out_arr) {
    FILE *f = fopen(LOG_FILE, "r");
    if (!f) return 0;
    char line[512];
    /* skip header */
    if (fgets(line, sizeof(line), f) == NULL) { fclose(f); return 0; }

    DayEntry *arr = NULL;
    int cap = 0, n = 0;
    while (fgets(line, sizeof(line), f)) {
        if (n + 1 > cap) {
            cap = cap ? cap*2 : 64;
            arr = realloc(arr, cap * sizeof(DayEntry));
            if (!arr) { fclose(f); return 0; }
        }
        DayEntry d;
        memset(&d, 0, sizeof(d));
        /* parse CSV line */
        /* Use sscanf and manual parsing to be robust */
        char *tok;
        char *rest = line;
        int field = 0;

        tok = strtok(rest, ",");
        while (tok) {
            switch (field) {
                case 0: strncpy(d.date, tok, sizeof(d.date)); d.date[10] = '\0'; break;
                case 1: d.car_km = atof(tok); break;
                case 2: d.bus_km = atof(tok); break;
                case 3: d.train_km = atof(tok); break;
                case 4: d.flight_km = atof(tok); break;
                case 5: d.electricity_kwh = atof(tok); break;
                case 6: d.plastic_kg = atof(tok); break;
                case 7: d.meat_meals = atoi(tok); break;
                case 8: d.veg_meals = atoi(tok); break;
                case 9: d.fish_meals = atoi(tok); break;
                case 10: d.recycled = atoi(tok); break;
                case 11: d.used_public_transport_count = atoi(tok); break;
                case 12: d.saved_electricity_actions = atoi(tok); break;
                case 13: d.footprint_kg = atof(tok); break;
                default: break;
            }
            tok = strtok(NULL, ",");
            field++;
        }
        arr[n++] = d;
    }
    fclose(f);
    *out_arr = arr;
    return n;
}

/* ---------------------------
   Visual / UI helpers
   --------------------------- */

/* Print an ASCII horizontal bar proportional to value/max_value */
void ascii_bar(double value, double max_value) {
    int width = MAX_BAR_WIDTH;
    int filled = 0;
    if (max_value > 0.0) {
        double ratio = value / max_value;
        if (ratio < 0) ratio = 0;
        if (ratio > 1) ratio = 1;
        filled = (int)(ratio * width + 0.5);
    }
    /* choose color based on value thresholds compared to max thresholds */
    const char *color = COLOR_GREEN;
    if (value > MEDIUM_THRESHOLD) color = COLOR_RED;
    else if (value > LOW_THRESHOLD) color = COLOR_YELLOW;

    printf("%s[", color);
    for (int i = 0; i < filled; ++i) putchar('=');
    for (int i = filled; i < width; ++i) putchar(' ');
    printf("]%s\n", COLOR_RESET);
}

/* Display a colored status line for a footprint */
void print_footprint_status(double kg) {
    const char *color = COLOR_GREEN;
    const char *label = "LOW";
    if (kg > MEDIUM_THRESHOLD) { color = COLOR_RED; label = "HIGH"; }
    else if (kg > LOW_THRESHOLD) { color = COLOR_YELLOW; label = "MEDIUM"; }
    printf("%sDaily footprint: %.2f kg CO2eq — [%s]%s\n", color, kg, label, COLOR_RESET);
}

/* Simple mini pie-like breakdown using percentages and ASCII bars for contributors */
void show_breakdown(DayEntry *d) {
    double car = d->car_km * EF_CAR_PER_KM;
    double bus = d->bus_km * EF_BUS_PER_KM;
    double train = d->train_km * EF_TRAIN_PER_KM;
    double flight = d->flight_km * EF_FLIGHT_PER_KM;
    double electricity = d->electricity_kwh * EF_ELECTRICITY_KWH;
    double plastic = d->plastic_kg * EF_PLASTIC_PER_KG;
    double food = d->meat_meals * EF_MEAT_MEAL + d->veg_meals * EF_VEG_MEAL + d->fish_meals * EF_FISH_MEAL;
    double parts[7] = {car, bus+train, flight, electricity, plastic, food, 0.0};
    const char *names[7] = {"Car", "Transit", "Flights", "Electricity", "Plastic", "Food", "Other"};

    double total = d->footprint_kg;
    if (total <= 0.0) {
        printf("No emissions recorded for this day.\n");
        return;
    }
    printf("\nBreakdown (approx):\n");
    double maxP = 0.0;
    for (int i = 0; i < 6; ++i) { if (parts[i] > maxP) maxP = parts[i]; }
    for (int i = 0; i < 6; ++i) {
        double val = parts[i];
        double pct = (val / total) * 100.0;
        printf(" %-11s %6.2f kg (%5.1f%%) ", names[i], val, pct);
        ascii_bar(val, maxP > 0 ? maxP : 1.0);
    }
    printf("\nTop contributor: %s\n", top_contributor(d));
}

/* Show suggestions based on inputs */
void show_suggestions(DayEntry *d) {
    printf("\nSuggestions to reduce your footprint (simple, practical):\n");
    const char *top = top_contributor(d);

    /* Generic suggestions */
    printf(" - Switch lights to LEDs and unplug idle chargers; use energy-saving settings for appliances.\n");
    printf(" - Carry a reusable water bottle and shopping bag to reduce single-use plastic.\n");
    printf(" - Try one or two plant-based meals weekly to cut food emissions.\n");
    printf(" - Combine trips, carpool, walk or cycle for short distances.\n");

    /* Specific suggestion when one area dominates */
    if (strcmp(top, "Car travel") == 0) {
        printf(COLOR_CYAN "  -> Since car travel is a major contributor, try: public transit, carpooling, biking,\n     or switching to a more efficient vehicle.\n" COLOR_RESET);
    } else if (strcmp(top, "Public transit") == 0) {
        printf(COLOR_CYAN "  -> Your public transit usage is significant (this is usually positive). Keep using buses/trains: they are often lower per-passenger.\n" COLOR_RESET);
    } else if (strcmp(top, "Flights") == 0) {
        printf(COLOR_CYAN "  -> Flights are high-impact. Consider fewer flights, longer stays per trip, or carbon offset programs.\n" COLOR_RESET);
    } else if (strcmp(top, "Electricity") == 0) {
        printf(COLOR_CYAN "  -> Electricity is a top contributor. Improve home insulation, reduce standby power, and shift heavy usage to off-peak times if possible.\n" COLOR_RESET);
    } else if (strcmp(top, "Plastic") == 0) {
        printf(COLOR_CYAN "  -> Plastic shows up strongly. Reduce single-use plastics, recycle, and choose products with less packaging.\n" COLOR_RESET);
    } else if (strcmp(top, "Food (meals)") == 0) {
        printf(COLOR_CYAN "  -> Food choices matter. Reduce portion of red meat, choose seasonal/local produce, and avoid food waste.\n" COLOR_RESET);
    }
    printf("\nSmall daily habits add up — even 10%% lower consumption in one area is meaningful.\n");
}

/* ---------------------------
   Higher-level statistics: daily, weekly, monthly
   --------------------------- */

/* Helper to compute sums over last N days (including today) */
void compute_stats_last_n_days(int n_days, double *out_sum, int *out_count_days_with_entries,
                               double *out_avg, int *out_recycle_count, int *out_public_transport_actions, int *out_saved_elec_actions) {
    DayEntry *arr = NULL;
    int n = read_all_entries(&arr);
    *out_sum = 0.0;
    *out_count_days_with_entries = 0;
    *out_recycle_count = 0;
    *out_public_transport_actions = 0;
    *out_saved_elec_actions = 0;

    if (n == 0) {
        if (arr) free(arr);
        *out_avg = 0.0;
        return;
    }
    time_t now = time(NULL);
    /* floor to midnight for today's time */
    struct tm tm_now = *localtime(&now);
    tm_now.tm_hour = 0; tm_now.tm_min = 0; tm_now.tm_sec = 0; tm_now.tm_isdst = -1;
    time_t today_midnight = mktime(&tm_now);

    for (int i = 0; i < n; ++i) {
        time_t t = parse_date_midnight(arr[i].date);
        if (t == (time_t)-1) continue;
        double diff_days = difftime(today_midnight, t) / (60*60*24);
        if (diff_days >= 0 && diff_days < n_days) {
            *out_sum += arr[i].footprint_kg;
            (*out_count_days_with_entries)++;
            *out_recycle_count += arr[i].recycled;
            *out_public_transport_actions += arr[i].used_public_transport_count;
            *out_saved_elec_actions += arr[i].saved_electricity_actions;
        }
    }
    if (*out_count_days_with_entries > 0) *out_avg = (*out_sum) / (*out_count_days_with_entries);
    else *out_avg = 0.0;
    free(arr);
}

/* Print a quick stats overview */
void show_overview() {
    double sum7, sum30;
    int days7, days30;
    double avg7, avg30;
    int recycle7, recycle30, pub7, pub30, saved7, saved30;

    compute_stats_last_n_days(7, &sum7, &days7, &avg7, &recycle7, &pub7, &saved7);
    compute_stats_last_n_days(30, &sum30, &days30, &avg30, &recycle30, &pub30, &saved30);

    printf("\n%s=== Quick Overview ===%s\n", COLOR_CYAN, COLOR_RESET);
    printf(" Last 7 days:  Total %.2f kg  Days recorded %d  Avg/day %.2f kg\n", sum7, days7, avg7);
    printf(" Last 30 days: Total %.2f kg  Days recorded %d  Avg/day %.2f kg\n", sum30, days30, avg30);
    printf("\nHabits (last 30 days): Recycle on %d days; public transport actions %d; saved-electricity actions %d\n",
           recycle30, pub30, saved30);
}

/* Print history lines (most recent first, limit optional) */
void show_history(int limit) {
    DayEntry *arr = NULL;
    int n = read_all_entries(&arr);
    if (n == 0) {
        printf("No history yet. Add your first entry!\n");
        return;
    }
    printf("\n%s=== History (latest first) ===%s\n", COLOR_CYAN, COLOR_RESET);
    int start = n - 1;
    int printed = 0;
    for (int i = start; i >= 0 && (limit <= 0 ? 1 : printed < limit); --i) {
        DayEntry *d = &arr[i];
        printf(" %s | %.2f kg CO2eq | car %.1f km | elec %.1f kWh | plastic %.2f kg | meals m:%d v:%d f:%d | recycle:%s\n",
               d->date, d->footprint_kg,
               d->car_km, d->electricity_kwh, d->plastic_kg,
               d->meat_meals, d->veg_meals, d->fish_meals,
               d->recycled ? "yes" : "no");
        printed++;
    }
    free(arr);
}

/* ---------------------------
   User input flow / menu actions
   --------------------------- */

void action_add_today_entry() {
    DayEntry d;
    memset(&d, 0, sizeof(d));
    get_today_str(d.date, sizeof(d.date));
    printf("\nEnter today's activities for %s (press Enter for suggested default shown):\n", d.date);

    d.car_km = prompt_double("Car travel distance (km)", 0.0);
    d.bus_km = prompt_double("Bus distance (km)", 0.0);
    d.train_km = prompt_double("Train distance (km)", 0.0);
    d.flight_km = prompt_double("Flights distance (km, total today)", 0.0);
    d.electricity_kwh = prompt_double("Electricity used today (kWh)", 2.0);
    d.plastic_kg = prompt_double("Plastic consumed today (kg of single-use items)", 0.0);
    d.meat_meals = prompt_int("Number of meat-heavy meals today", 0);
    d.veg_meals = prompt_int("Number of vegetarian meals today", 0);
    d.fish_meals = prompt_int("Number of fish meals today", 0);
    d.recycled = prompt_yesno("Did you recycle today?", 1);
    d.used_public_transport_count = prompt_int("Number of times you used public transport today (each ride counted)", 0);
    d.saved_electricity_actions = prompt_int("Count electricity-saving actions you did today (e.g., unplugged, turned off lights)", 0);

    calculate_footprint(&d);
    printf("\n%sRecorded for %s: %.2f kg CO2eq%s\n", COLOR_CYAN, d.date, d.footprint_kg, COLOR_RESET);
    print_footprint_status(d.footprint_kg);
    show_breakdown(&d);
    show_suggestions(&d);

    if (prompt_yesno("Save this entry to local log file?", 1)) {
        if (append_entry_to_log(&d)) {
            printf("%sSaved to %s%s\n", COLOR_GREEN, LOG_FILE, COLOR_RESET);
        } else {
            printf("%sFailed to save.%s\n", COLOR_RED, COLOR_RESET);
        }
    } else {
        printf("Entry not saved.\n");
    }
}

void action_view_today() {
    char today[16];
    get_today_str(today, sizeof(today));
    DayEntry *arr = NULL;
    int n = read_all_entries(&arr);
    if (n == 0) {
        printf("No entries found. Add today's entry first.\n");
        return;
    }
    for (int i = 0; i < n; ++i) {
        if (strcmp(arr[i].date, today) == 0) {
            printf("\nEntry for today (%s):\n", today);
            printf(" Footprint: %.2f kg CO2eq\n", arr[i].footprint_kg);
            print_footprint_status(arr[i].footprint_kg);
            show_breakdown(&arr[i]);
            show_suggestions(&arr[i]);
            free(arr);
            return;
        }
    }
    free(arr);
    printf("No entry for today yet. Would you like to add one?\n");
    if (prompt_yesno("Add today's entry now?", 1)) action_add_today_entry();
}

void action_view_stats() {
    show_overview();
    double sum7, sum30;
    int days7, days30;
    double avg7, avg30;
    int recycle7, recycle30, pub7, pub30, saved7, saved30;

    compute_stats_last_n_days(7, &sum7, &days7, &avg7, &recycle7, &pub7, &saved7);
    compute_stats_last_n_days(30, &sum30, &days30, &avg30, &recycle30, &pub30, &saved30);

    /* visual: show average day bar */
    printf("\nVisual: average per day (last 7 days):\n");
    ascii_bar(avg7, MEDIUM_THRESHOLD*2);
    printf("Visual: average per day (last 30 days):\n");
    ascii_bar(avg30, MEDIUM_THRESHOLD*2);
}

/* Simple habit encouragements and summary */
void action_habits() {
    double sum30;
    int days30;
    double avg30;
    int recycle30, pub30, saved30;
    compute_stats_last_n_days(30, &sum30, &days30, &avg30, &recycle30, &pub30, &saved30);

    printf("\n%s=== Habit Tracker (last 30 days) ===%s\n", COLOR_CYAN, COLOR_RESET);
    if (days30 == 0) {
        printf("No data in last 30 days. Start logging daily — it helps learning!\n");
        return;
    }
    printf("Days recorded: %d\n", days30);
    printf("Recycled on %d days (%.0f%%)\n", recycle30, (double)recycle30/days30*100.0);
    printf("Public transport actions (total rides): %d\n", pub30);
    printf("Saved-electricity actions (total): %d\n", saved30);

    printf("\nSimple habit goals suggestions:\n");
    printf(" - Aim to recycle at least 50%% of recorded days.\n");
    printf(" - Try to replace 1 car trip per week with walking, cycling or public transport.\n");
    printf(" - Plan 1-2 meat-free days each week to reduce food emissions.\n");
}

/* Allow user to export a summary text file */
void action_export_report() {
    char filename[128];
    char defaultname[64];
    time_t now = time(NULL);
    struct tm tm = *localtime(&now);
    snprintf(defaultname, sizeof(defaultname), "eco_report_%04d%02d%02d.txt", tm.tm_year+1900, tm.tm_mon+1, tm.tm_mday);
    printf("Enter filename to export (default %s): ", defaultname);
    input_line(filename, sizeof(filename));
    if (strlen(filename) == 0) strncpy(filename, defaultname, sizeof(filename));

    DayEntry *arr = NULL;
    int n = read_all_entries(&arr);
    FILE *f = fopen(filename, "w");
    if (!f) { printf("Could not open %s for writing.\n", filename); if (arr) free(arr); return; }

    fprintf(f, "Eco Tracker Report - generated %04d-%02d-%02d\n\n", tm.tm_year+1900, tm.tm_mon+1, tm.tm_mday);
    if (n == 0) {
        fprintf(f, "No data\n");
        fclose(f);
        if (arr) free(arr);
        printf("Exported empty report to %s\n", filename);
        return;
    }
    double total_all = 0;
    for (int i = 0; i < n; ++i) {
        DayEntry *d = &arr[i];
        fprintf(f, "%s , %.2f kg CO2\n", d->date, d->footprint_kg);
        total_all += d->footprint_kg;
    }
    fprintf(f, "\nTotal across %d entries: %.2f kg CO2\n", n, total_all);
    fclose(f);
    if (arr) free(arr);
    printf("Exported report to %s\n", filename);
}

/* ---------------------------
   Main menu & loop
   --------------------------- */

void show_main_menu() {
    printf("\n%s=== Eco Tracker (Beginner-friendly) ===%s\n", COLOR_CYAN, COLOR_RESET);
    printf("1) Add today's entry (quick & friendly)\n");
    printf("2) View today's footprint & suggestions\n");
    printf("3) View stats (daily / weekly / monthly)\n");
    printf("4) Habit tracker summary\n");
    printf("5) History (latest entries)\n");
    printf("6) Export a simple report to text file\n");
    printf("7) Help / About\n");
    printf("0) Exit\n");
    printf("Choose an option: ");
}

void show_help() {
    printf("\nHelp & Quick Tips:\n");
    printf(" - Keep entries short: only count what you actually used today.\n");
    printf(" - Electricity: enter kWh from your meter or estimate (e.g., small room ~0.5-1 kWh per hour).\n");
    printf(" - Travel: enter kilometers for each mode (car, bus, train). Short trips matter.\n");
    printf(" - Food: count meals (meat/veg/fish) you had today.\n");
    printf(" - The app stores entries locally in '%s' as CSV.\n", LOG_FILE);
    printf(" - Visual bars and color codes show low/medium/high impact.\n");
    printf(" - Suggestions are educational — try one change per week and watch your average improve!\n");
}

/* ---------------------------
   Entry point
   --------------------------- */

int main(void) {
    ensure_log_header();
    int choice = -1;
    char line[32];
    while (1) {
        show_main_menu();
        input_line(line, sizeof(line));
        if (strlen(line) == 0) continue;
        choice = atoi(line);
        switch (choice) {
            case 1: action_add_today_entry(); break;
            case 2: action_view_today(); break;
            case 3: action_view_stats(); break;
            case 4: action_habits(); break;
            case 5:
                show_history(20);
                break;
            case 6:
                action_export_report();
                break;
            case 7:
                show_help();
                break;
            case 0:
                printf("Goodbye — keep making small changes! 🌱\n");
                return 0;
            default:
                printf("Invalid option. Try again.\n");
        }
    }
    return 0;
}

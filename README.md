# Basic-Data-Cleaning
![Cleaning Bubbles](https://imagehost9.online-image-editor.com/oie_upload/images/723252a8Gs5X/TKXs04Ia51Nw.png)
##Turn messy data into clean, ready-to-use datasets
This project takes a messy dataset full of duplicates, weird formatting, missing data, and blank fields — and turns it into something polished and analysis-ready.

🔥 What I Did:

Cloned the raw table to work safely

Zapped duplicates using ROW_NUMBER()

Cleaned messy company names, industries, and country fields

Converted janky string dates to real SQL DATEs

Filled in missing industries by matching companies

Removed empty/useless records

Ended with a lean, clean dataset ready for insight


-- 💾 1. Backup the Mess — "Don't break the original!"
-- We make a full copy of the raw data first — think of it like working on a clone so the original stays safe.
CREATE TABLE layoffs_staging LIKE layoffs;
INSERT layoffs_staging SELECT * FROM layoffs;

-- 🔍 2. Hunt Down Duplicates — "The evil twins"
-- We use ROW_NUMBER to assign a number to each row within identical groups.
-- Anything with row_num > 1 is a duplicate!
WITH duplicate_cte AS (
  SELECT *, 
         ROW_NUMBER() OVER (
           PARTITION BY company, location, industry, total_laid_off,
                        percentage_laid_off, `date`, stage, country,
                        funds_raised_millions) AS row_num
  FROM layoffs_staging
)
SELECT * FROM duplicate_cte WHERE row_num > 1;

-- 🧼 3. Build a Clean Table — "New home, fresh start"
-- Now we insert into a new staging table WITH the row numbers so we can easily delete the duplicates.
CREATE TABLE layoffs_staging2 (
  company TEXT,
  location TEXT,
  industry TEXT,
  total_laid_off INT DEFAULT NULL,
  percentage_laid_off TEXT,
  date TEXT,
  stage TEXT,
  country TEXT,
  funds_raised_millions INT DEFAULT NULL,
  row_num INT
);

INSERT INTO layoffs_staging2
SELECT *, 
       ROW_NUMBER() OVER (
         PARTITION BY company, location, industry, total_laid_off,
                      percentage_laid_off, `date`, stage, country,
                      funds_raised_millions) AS row_num
FROM layoffs_staging;

-- 👋 Bye Bye Duplicates — "You're not welcome here!"
DELETE FROM layoffs_staging2 WHERE row_num > 1;

-- 🧹 4. Clean Up Names — "No more messy strings"
-- Trims sneaky spaces and standardizes inconsistent entries.
UPDATE layoffs_staging2 SET company = TRIM(company);

-- Let's keep all things "crypto" under one name.
UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE '%crypto%';

-- Clean trailing periods on country names — "Punctuation police!"
UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE '%united states%';

-- 📅 5. Fix the Dates — "From string soup to calendar-ready"
UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');
ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;

-- 🚫 6. Null the Blanks — "Say it with NULLs"
-- Blank industry fields are turned into proper NULLs for better handling.
UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';

-- 🧠 7. Fill in the Gaps — "Borrow from your neighbors"
-- If some company rows are missing industry info, borrow it from others that have it.
UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2 ON t1.company = t2.company
SET t1.industry = t2.industry
WHERE (t1.industry IS NULL OR t1.industry = '')
  AND t2.industry IS NOT NULL;

-- 🗑️ 8. Remove the Trash — "No info? No space for you!"
-- Drop rows that tell us absolutely nothing about layoffs.
DELETE FROM layoffs_staging2
WHERE total_laid_off IS NULL
  AND percentage_laid_off IS NULL;

-- 🧽 9. Final Touch — "Wipe off that extra column"
ALTER TABLE layoffs_staging2
DROP COLUMN row_num;


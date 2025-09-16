---
{"dg-publish":true,"permalink":"/system/Macros/End_phase/","noteIcon":"","created":"2025-09-16T23:43:16.943+03:00","updated":"2025-09-16T23:52:45.447+03:00"}
---


<%*
const file = tp.file.find_tfile(tp.file.title);
const oldContent = await app.vault.read(file);
// 1. Сохраняем старый контент
//const oldContentCopy = oldContent.trim();


// 2. Разбиваем на блоки персонажей
const contentBeforeEnd = oldContent.split('---')[0];
const blocks = contentBeforeEnd.split(/(?=<\[\[)/g)
  .map(b => b.trim())
 // .filter(b => b.startsWith('<[['));


// 3. Считываем ход и увеличиваем на 1
let moveMatch = oldContent.match(/\[\[Раунд\]\]:\s*(\d+)/);
let moveValue = moveMatch ? Number(moveMatch[1]) + 1 : 1;


// 4. Парсим данные по каждому персонажу
let result = [];
blocks.forEach(block => {
  const nameMatch = block.match(/^<\[\[([^\]]+)\]\]>/);
  if (!nameMatch) return;
  const name = nameMatch[1];
  const characteristics = {};
  const statusesWithPart = [];

  console.log(`=== Парсинг персонажа: ${name} ===`);

  
  // Основные характеристики
  const regexSimple = /\[\[([^\]]+)\]\]:\s*(-?\d+)(?:\s*\((\d+)\))?/g;
  let match;
  while ((match = regexSimple.exec(block)) !== null) {
    let key = match[1];
    let current = Number(match[2]);
    let max = match[3] ? Number(match[3]) : current;
    characteristics[key] = { current, max };
    console.log(`Характеристика: ${key} = ${current} (max: ${max})`);
  }


  // Статусы с частью тела
  const lines = block.split(',');
  for (const line of lines) {
    // Удаляем эмодзи перед парсингом
    const cleanLine = line.replace(/[🟥🟦]/g, '').trim();
    const statusMatch = cleanLine.match(/^\[\[(Увечье|Иней|Ожог|Экватор)\]\]:\s*(-?\d+)\s*-\s*([^,]+)$/);
    if (statusMatch) {
      statusesWithPart.push({ 
        type: statusMatch[1], 
        duration: Number(statusMatch[2]),
        part: statusMatch[3].trim()
      });
      console.log(`Статус: ${statusMatch[1]} = ${statusMatch[2]} - ${statusMatch[3].trim()}`);
    } else if (cleanLine.includes('Иней') || cleanLine.includes('Ожог') || cleanLine.includes('Увечье') || cleanLine.includes('Экватор')) {
      console.log(`Не распознана строка после очистки: "${cleanLine}"`);
    }
  }

  result.push({ name, characteristics, statusesWithPart });
  console.log(`Все статусы для ${name}:`, statusesWithPart);
});

// -------------------------------------- Начало обработки --------------------------------------
// Обработка каждого персонажа
result.forEach(character => {
  console.log(`\n=== Обработка: ${character.name} ===`);
  let c = character.characteristics;

 
  // === Обработка Экватор ===
  let contrastStatus = character.statusesWithPart.find(s => s.type === 'Экватор');
  let frostStatus = character.statusesWithPart.find(s => s.type === 'Иней');
  let burnStatus = character.statusesWithPart.find(s => s.type === 'Ожог');
  
  let contrastValue = 0;
  let contrastPart = '';
  
  console.log(`Начальный Экватор:`, contrastStatus);
  console.log(`Иней:`, frostStatus);
  console.log(`Ожог:`, burnStatus);
  
  // Если есть текущий Экватор, берем его значение
  if (contrastStatus) {
    contrastValue = contrastStatus.duration;
    contrastPart = contrastStatus.part;
    console.log(`Берём существующий Экватор: ${contrastValue} - ${contrastPart}`);
  }
  
  // Обработка Ожога
  if (burnStatus && burnStatus.duration > 0) {
    const burnValue = burnStatus.duration;
    let contrastChange = 0;
    
    if (contrastValue < 1) contrastChange = 1 + burnValue;
    else if (contrastValue >= 1 && contrastValue <= 5) contrastChange = 2 + burnValue;
    else contrastChange = 3 * burnValue;
    
    contrastValue += contrastChange;
    contrastPart = burnStatus.part;
    console.log(`Ожог ${burnValue} -> Экватор +${contrastChange} = ${contrastValue} (часть: ${contrastPart})`);
  }
  
  // Обработка Инея
  if (frostStatus && frostStatus.duration > 0) {
    const frostValue = frostStatus.duration;
    let contrastChange = 0;
    
    if (contrastValue > -1) contrastChange = -1 - frostValue;
    else if (contrastValue <= -1 && contrastValue >= -5) contrastChange = -2 - frostValue;
    else contrastChange = -3 - frostValue;
    
    contrastValue += contrastChange;
    contrastPart = frostStatus.part;
    console.log(`Иней ${frostValue} -> Экватор ${contrastChange} = ${contrastValue} (часть: ${contrastPart})`);
  }
  
  // Экватор приближается к нулю
  if (contrastValue > 0) {
    contrastValue -= 1;
    console.log(`Экватор приближается к 0: -1 = ${contrastValue}`);
  } else if (contrastValue < 0) {
    contrastValue += 1;
    console.log(`Экватор приближается к 0: +1 = ${contrastValue}`);
  }
  
  // Урон от Экватора
  if (contrastValue !== 0) {
    let damage = Math.floor(Math.abs(contrastValue) / 3 + 1);
    console.log(`Урон от Экватора ${contrastValue}: damage = ${damage}`);
    
    if (damage > 0) {
      if (contrastValue < 0) { // Охлаждение
        console.log("Охлаждение - урон по барьеру -> здоровью -> щиту");
        // сначала барьер
        if (c["Барьер"] && c["Барьер"].current > 0) {
          let dmg = Math.min(damage, c["Барьер"].current);
          c["Барьер"].current -= dmg;
          damage -= dmg;
          console.log(`Барьер: -${dmg} = ${c["Барьер"].current}, осталось damage: ${damage}`);
        }
        // потом здоровье
        if (damage > 0 && c["Здоровье"] && c["Здоровье"].current > 0) {
          let dmg = Math.min(damage, c["Здоровье"].current);
          c["Здоровье"].current -= dmg;
          damage -= dmg;
          console.log(`Здоровье: -${dmg} = ${c["Здоровье"].current}, осталось damage: ${damage}`);
        }
        // потом щит
        if (damage > 0 && c["Щит"] && c["Щит"].current > 0) {
          let dmg = Math.min(damage, c["Щит"].current);
          c["Щит"].current -= dmg;
          console.log(`Щит: -${dmg} = ${c["Щит"].current}`);
        }
      } else { // Перегрев
        console.log("Перегрев - урон по барьеру -> здоровью -> щиту");
        // сначала барьер
        if (c["Барьер"] && c["Барьер"].current > 0) {
          let dmg = Math.min(damage, c["Барьер"].current);
          c["Барьер"].current -= dmg;
          damage -= dmg;
          console.log(`Барьер: -${dmg} = ${c["Барьер"].current}, осталось damage: ${damage}`);
        }
        // потом здоровье
        if (damage > 0 && c["Здоровье"] && c["Здоровье"].current > 0) {
          let dmg = Math.min(damage, c["Здоровье"].current);
          c["Здоровье"].current -= dmg;
          damage -= dmg;
          console.log(`Здоровье: -${dmg} = ${c["Здоровье"].current}, осталось damage: ${damage}`);
        }
        // потом щит
        if (damage > 0 && c["Щит"] && c["Щит"].current > 0) {
          let dmg = Math.min(damage, c["Щит"].current);
          c["Щит"].current -= dmg;
          console.log(`Щит: -${dmg} = ${c["Щит"].current}`);
        }
      }
    }
    
    // Обновляем или добавляем Экватор

      const existingContrastIndex = character.statusesWithPart.findIndex(s => s.type === 'Экватор');
      if (existingContrastIndex !== -1) {
        character.statusesWithPart[existingContrastIndex].duration = contrastValue;
        if (!character.statusesWithPart[existingContrastIndex].part && contrastPart) {
          character.statusesWithPart[existingContrastIndex].part = contrastPart;
        }
        console.log(`Обновлён Экватор: ${contrastValue} - ${character.statusesWithPart[existingContrastIndex].part}`);
      } else {
        const finalPart = contrastPart || burnStatus?.part || frostStatus?.part || 'тело';
        character.statusesWithPart.push({ 
          type: 'Экватор', 
          part: finalPart, 
          duration: contrastValue 
        });
        console.log(`Добавлен новый Экватор: ${contrastValue} - ${finalPart}`);
      }
    } else {
      character.statusesWithPart = character.statusesWithPart.filter(s => s.type !== 'Экватор');
      console.log("Экватор удалён (достиг нуля)");
    }
  

 // === Отрава ===
  let poison = c["Отрава"]?.current ?? 0;
  if (poison > 0) {
    // сначала барьер
    if (c["Щит"]) {
      let dmg = Math.min(poison, c["Щит"].current);
      c["Щит"].current -= dmg;
      poison -= dmg;
    }
    // потом здоровье
    if (poison > 0 && c["Здоровье"]) {
      let dmg = Math.min(poison, c["Здоровье"].current);
      c["Здоровье"].current -= dmg;
      poison -= dmg;
    }
    // потом щит
    if (poison > 0 && c["Барьер"]) {
      let dmg = Math.min(poison, c["Барьер"].current);
      c["Барьер"].current -= dmg;
      poison -= dmg;
    }

  }
  

  // === Эфир ===
  let ether = c["Эфир"]?.current ?? 0;
  if (ether > 0) {
    // сначала барьер
    if (c["Барьер"]) {
      let dmg = Math.min(ether, c["Барьер"].current);
      c["Барьер"].current -= dmg;
      ether -= dmg;
    }
    // потом здоровье
    if (ether > 0 && c["Здоровье"]) {
      let dmg = Math.min(ether, c["Здоровье"].current);
      c["Здоровье"].current -= dmg;
      ether -= dmg;
    }
    // потом щит
    if (ether > 0 && c["Щит"]) {
      let dmg = Math.min(ether, c["Щит"].current);
      c["Щит"].current -= dmg;
      ether -= dmg;
    }
  }


      // === Грёзы уменьшают максимум ===
  if (c["Грёзы"]) {
    let dreams = c["Грёзы"].current;
    ["Барьер","Щит","Здоровье"].forEach(stat => {
      if (dreams > 0 && c[stat]) {
        let reduce = Math.min(dreams, c[stat].max);
        c[stat].max -= reduce;
        if (c[stat].current > c[stat].max) c[stat].current = c[stat].max;
        dreams -= reduce;
      }
    });
    delete c["Грёзы"]; // больше не выводим
  }


// === Жизнетик ===
if (c["Жизнетик"] && c["Жизнетик"].current > 0) {
  let lifeTick = c["Жизнетик"].current;
  let heal = Math.floor(lifeTick / 3 + 1);

  if (heal > 0) {
    // сначала здоровье
    if (c["Здоровье"] && c["Здоровье"].current < c["Здоровье"].max) {
      let rec = Math.min(heal, c["Здоровье"].max - c["Здоровье"].current);
      c["Здоровье"].current += rec;
      heal -= rec;
    }

    // потом щит
    if (heal > 0 && c["Щит"] && c["Щит"].current < c["Щит"].max) {
      let rec = Math.min(heal, c["Щит"].max - c["Щит"].current);
      c["Щит"].current += rec;
      heal -= rec;
    }

    // потом барьер
    if (heal > 0 && c["Барьер"] && c["Барьер"].current < c["Барьер"].max) {
      let rec = Math.min(heal, c["Барьер"].max - c["Барьер"].current);
      c["Барьер"].current += rec;
      heal -= rec;
    }
  }

  // уменьшаем Жизнетик на 1
  c["Жизнетик"].current = Math.max(0, lifeTick - 1);
}


// === Оклятие ===
if (c["Оклятие"] && c["Оклятие"].current > 0) {
  let heal = c["Оклятие"].current;

  if (c["Барьер"] && c["Барьер"].current < c["Барьер"].max) {
    let rec = Math.min(heal, c["Барьер"].max - c["Барьер"].current);
    c["Барьер"].current += rec;
    heal -= rec;
  }

  if (heal > 0 && c["Здоровье"] && c["Здоровье"].current < c["Здоровье"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Здоровье"].max - c["Здоровье"].current);
    c["Здоровье"].current += rec;
    heal -= rec * 2;
  }

  if (heal > 0 && c["Щит"] && c["Щит"].current < c["Щит"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Щит"].max - c["Щит"].current);
    c["Щит"].current += rec;
    heal -= rec * 2;
  }

  delete c["Оклятие"]; // исчезает после срабатывания
}


// === Укрепление ===
if (c["Укрепление"] && c["Укрепление"].current > 0) {
  let heal = c["Укрепление"].current;

  if (c["Щит"] && c["Щит"].current < c["Щит"].max) {
    let rec = Math.min(heal, c["Щит"].max - c["Щит"].current);
    c["Щит"].current += rec;
    heal -= rec;
  }

  if (heal > 0 && c["Здоровье"] && c["Здоровье"].current < c["Здоровье"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Здоровье"].max - c["Здоровье"].current);
    c["Здоровье"].current += rec;
    heal -= rec * 2;
  }

  if (heal > 0 && c["Барьер"] && c["Барьер"].current < c["Барьер"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Барьер"].max - c["Барьер"].current);
    c["Барьер"].current += rec;
    heal -= rec * 2;
  }

  delete c["Укрепление"]; // исчезает после срабатывания
}


// === Исцеление ===
if (c["Исцеление"] && c["Исцеление"].current > 0) {
  let heal = c["Исцеление"].current;

  if (c["Здоровье"] && c["Здоровье"].current < c["Здоровье"].max) {
    let rec = Math.min(heal, c["Здоровье"].max - c["Здоровье"].current);
    c["Здоровье"].current += rec;
    heal -= rec;
  }

  if (heal > 0 && c["Щит"] && c["Щит"].current < c["Щит"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Щит"].max - c["Щит"].current);
    c["Щит"].current += rec;
    heal -= rec * 2;
  }

  if (heal > 0 && c["Барьер"] && c["Барьер"].current < c["Барьер"].max) {
    let rec = Math.min(Math.floor(heal / 2), c["Барьер"].max - c["Барьер"].current);
    c["Барьер"].current += rec;
    heal -= rec * 2;
  }

  delete c["Исцеление"]; // исчезает после срабатывания
}


// === Гниение ===
if (c["Гниение"]) {
  let gnienie = c["Гниение"]?.current ?? 0;
  console.log("Значение Гниения = ", gnienie);

  // --- 1. Проверка разницы между max и current ---
  // Барьер
  if ((c["Барьер"]?.max ?? 0) - (c["Барьер"]?.current ?? 0) >= gnienie) {
    c["Барьер"].max -= gnienie;
    console.log("Снижен макс. Барьер на", gnienie, "=>", c["Барьер"]);
  }
  // Щит
  else if ((c["Щит"]?.max ?? 0) - (c["Щит"]?.current ?? 0) >= gnienie) {
    c["Щит"].max -= gnienie;
    console.log("Снижен макс. Щит на", gnienie, "=>", c["Щит"]);
  }
  // Здоровье
  else if ((c["Здоровье"]?.max ?? 0) - (c["Здоровье"]?.current ?? 0) >= gnienie) {
    c["Здоровье"].max -= gnienie;
    console.log("Снижено макс. Здоровье на", gnienie, "=>", c["Здоровье"]);
  }
  // --- 2. Если max не подходит — проверяем текущие ---
  else if ((c["Барьер"]?.current ?? 0) >= gnienie) {
    c["Барьер"].current -= gnienie;
    console.log("Снижен текущий Барьер на", gnienie, "=>", c["Барьер"]);
  }
  else if ((c["Щит"]?.current ?? 0) >= gnienie) {
    c["Щит"].current -= gnienie;
    console.log("Снижен текущий Щит на", gnienie, "=>", c["Щит"]);
  }
  else if ((c["Здоровье"]?.current ?? 0) >= gnienie) {
    c["Здоровье"].current -= gnienie;
    console.log("Снижено текущее Здоровье на", gnienie, "=>", c["Здоровье"]);
  }
  // --- 3. Если и это не сработало — «сжигаем» всё по цепочке ---
  else {
    let remaining = gnienie;

    // Барьер
    if (c["Барьер"]?.current > 0) {
      let take = Math.min(c["Барьер"].current, remaining);
      c["Барьер"].current -= take;
      remaining -= take;
      console.log("Сожжён Барьер на", take, "=>", c["Барьер"]);
    }

    // Щит
    if (remaining > 0 && c["Щит"]?.current > 0) {
      let take = Math.min(c["Щит"].current, remaining);
      c["Щит"].current -= take;
      remaining -= take;
      console.log("Сожжён Щит на", take, "=>", c["Щит"]);
    }

    // Здоровье
    if (remaining > 0 && c["Здоровье"]?.current > 0) {
      let take = Math.min(c["Здоровье"].current, remaining);
      c["Здоровье"].current -= take;
      remaining -= take;
      console.log("Сожжено Здоровье на", take, "=>", c["Здоровье"]);
    }

    console.log("Остаток после применения Гниения =", remaining);
  }
}



  

// === Закат / Пустота ===
if (c["Закат"] || c["Пустота"]) {
let zakat = c["Закат"]?.current ?? 0;
  let maxSum = (c["Здоровье"]?.max ?? 0) + (c["Барьер"]?.max ?? 0) + (c["Скорость"]?.max ?? 0) + (c["Щит"]?.max ?? 0);
  console.log("Сумма макс. характеристик равна ", maxSum);
  let curSum = (c["Здоровье"]?.current ?? 0) + (c["Барьер"]?.current ?? 0) + (c["Скорость"]?.current ?? 0) + (c["Щит"]?.current ?? 0);
  console.log("Сумма текущих характеристик равна ", curSum);
  let diff = maxSum - curSum;
  console.log("Разница (потерянные очки) равна ", diff);
  
  let newVoidValue = diff + zakat;
  console.log("Новое значение Пустоты (diff + zakat) = ", newVoidValue);

  if (!c["Пустота"]) {
    // Если Пустоты еще нет, создаем с новым значением
    c["Пустота"] = { current: newVoidValue, max: newVoidValue };
    console.log("Создана новая Пустота: ", c["Пустота"]);
  } else {
    // Сравниваем (текущая Пустота + Закат) и (diff + Закат), берем наибольшее
    let currentPlusZakat = c["Пустота"].current + zakat;
    let maxValue = Math.max(currentPlusZakat, newVoidValue);
    
    console.log("Текущая Пустота + Закат = ", currentPlusZakat);
    console.log("Max значение для сравнения = ", maxValue);
    
    c["Пустота"].current = maxValue;
    
    // Обновляем максимум если нужно
    if (c["Пустота"].current > c["Пустота"].max) {
      c["Пустота"].max = c["Пустота"].current;
    }
    console.log("Обновленная Пустота: ", c["Пустота"]);
  }
}


  // === Статусы уменьшаются ===
  ["Отрава","Ужас","Эфир", "Невезение", "Везение", "Длань возмездия", "Гниение", "Ярость"].forEach(stat => {
    if(c[stat]) c[stat].current = Math.max(0, c[stat].current - 1);
  });
  

  // Скорость и увечья
  let legInjury = character.statusesWithPart.find(
    s => s.type === 'Увечье' && s.part.toLowerCase() === 'ноги' && s.duration > 0
  );
  if (c["Скорость"]) {
    if (legInjury) {
      c["Скорость"].current = Math.max(0, c["Скорость"].max - legInjury.duration);
    } else {
      c["Скорость"].current = c["Скорость"].max;
    }
  }


// === Фортуна (с взаимоподавлением Везение <-> Невезение) ===
if (c["Фортуна"]) {
  let fortuna = c["Фортуна"]?.current ?? 0;
  console.log("Значение Фортуны = ", fortuna);

  // хелпер: применить value к targetName с учётом противоположного oppositeName
  function applyOpposingStatus(targetName, oppositeName, value) {
    if (!value || value <= 0) return;

    let oppCur = c[oppositeName]?.current ?? 0;

    // если есть противоположный эффект — пробуем его погасить
    if (oppCur > 0) {
      if (oppCur > value) {
        // противоположный больше — просто уменьшаем его
        c[oppositeName].current = oppCur - value;
        console.log(`${oppositeName} уменьшено на ${value} =>`, c[oppositeName]);
      } else {
        // противоположный меньше или равен — удаляем его, остаток может пойти в целевой
        let leftover = value - oppCur;
        // удаляем противоположный статус
        c[oppositeName].current = 0;
        delete c[oppositeName];
        console.log(`${oppositeName} снято (погашено ${oppCur})`);

        // если есть остаток — добавляем его к целевому (создав при необходимости)
        if (leftover > 0) {
          if (!c[targetName]) {
            c[targetName] = { current: leftover, max: leftover };
            console.log(`Создан ${targetName} с ${leftover}`);
          } else {
            c[targetName].current += leftover;
            c[targetName].max += leftover;
            console.log(`${targetName} увеличено на остаток ${leftover} =>`, c[targetName]);
          }
        } else {
          // leftover == 0 — ничего добавлять не нужно
          console.log("Остаток = 0, целевой статус не создаётся/не изменяется");
        }
      }
    } else {
      // противоположного нет — просто добавляем к целевому (создаём если надо)
      if (!c[targetName]) {
        c[targetName] = { current: value, max: value };
        console.log(`Создан ${targetName} с ${value}`);
      } else {
        c[targetName].current += value;
        c[targetName].max += value;
        console.log(`${targetName} увеличено на ${value} =>`, c[targetName]);
      }
    }
  }

  // Рандом 0 или 1
  let roll = Math.random() < 0.5 ? 0 : 1;
  console.log("Результат броска (0=невезение, 1=везение): ", roll);

  // --- Гниение (всегда) ---
  if (!c["Гниение"]) {
    c["Гниение"] = { current: fortuna, max: fortuna };
    console.log("Наложено новое Гниение:", c["Гниение"]);
  } else {
    c["Гниение"].current += fortuna;
    c["Гниение"].max += fortuna;
    console.log("Увеличено Гниение:", c["Гниение"]);
  }

  // --- Применяем Везение/Невезение с взаимоподавлением ---
  if (roll === 1) {
    // Везение vs Невезение
    applyOpposingStatus("Везение", "Невезение", fortuna);
  } else {
    // Невезение vs Везение
    applyOpposingStatus("Невезение", "Везение", fortuna);
  }
}


  // Уменьшаем длительность статусов (кроме Экватора)
  character.statusesWithPart.forEach(s => {
    if (s.type !== 'Экватор') {
      const oldDuration = s.duration;
      s.duration = Math.max(-1, s.duration - 1);
      console.log(`Статус ${s.type} уменьшен: ${oldDuration} -> ${s.duration}`);
    }
  });
  
  // Удаляем статусы с нулевой или отрицательной длительностью (кроме Экватора)
  const beforeFilter = character.statusesWithPart.length;
  character.statusesWithPart = character.statusesWithPart.filter(s => 
    s.duration > 0 || s.type === 'Экватор'
  );
  console.log(`Статусов после фильтрации: ${beforeFilter} -> ${character.statusesWithPart.length}`);
});

// === Формируем вывод ===
let output=`[[Раунд]]: ${moveValue}\n\n`;

for(let character of result){
  let chars=character.characteristics;

  output+=`<[[${character.name}]]>:\n`;

  output+=`[[Здоровье]]: ${chars['Здоровье']?.current ?? 0} (${chars['Здоровье']?.max ?? 0}), `;
  output+=`[[Барьер]]: ${chars['Барьер']?.current ?? 0} (${chars['Барьер']?.max ?? 0}), `;
  output+=`[[Щит]]: ${chars['Щит']?.current ?? 0} (${chars['Щит']?.max ?? 0}), `;
  output+=`[[Скорость]]: ${chars['Скорость']?.current ?? 0} (${chars['Скорость']?.max ?? 0}), \n\n`;

  // Статусы с частью тела
  let statusParts=[];
  for(let statName of ["Увечье","Экватор"]) {
    character.statusesWithPart.filter(s=>s.type===statName)
    .forEach(s=>{
      let statusText = `[[${statName}]]: ${s.duration} - ${s.part}`;
      
      // Добавляем эмодзи для Экватора
      if (statName === 'Экватор') {
        const damage = Math.floor(Math.abs(s.duration) / 3 + 1);
        if (s.duration > 0) {
          statusText += ' ' + '🟥'.repeat(damage);
        } else if (s.duration < 0) {
          statusText += ' ' + '🟦'.repeat(damage);
        }

      }
	  

      statusParts.push(statusText + ',');
    });
  }
  if (statusParts.length > 0) {
    output += statusParts.join(", ") + "\n";
  }

  // Остальные статусы
  let statusSimple=[];
  for(let stat of ["Поток","Отрава","Ужас","Эфир","Пустота", "Невезение", "Везение", "Длань возмездия", "Жизнетик", "Ледяной щит", "Гниение", "Ярость"]) {
    if(chars[stat]?.current>0)
      statusSimple.push(`[[${stat}]]: ${chars[stat].current}`);
  }
  if(statusSimple.length>0) output+=statusSimple.join(", ")+ "\n";

  output+="  \n\n";
}

output+="---\n"+oldContent;

await app.vault.modify(file, '');
tR+=output;
tp.file.cursor(0);
%>
🔹 Merge Sort (Сортировка слиянием)

```csharp
using System;

class Program
{
    static void MergeSort(int[] arr, int left, int right)
    {
        if (left < right)
        {
            int mid = (left + right) / 2;

            // Рекурсивно делим массив на 2 части
            MergeSort(arr, left, mid);
            MergeSort(arr, mid + 1, right);

            // Потом сливаем отсортированные половины
            Merge(arr, left, mid, right);
        }
    }

    static void Merge(int[] arr, int left, int mid, int right)
    {
        int n1 = mid - left + 1;
        int n2 = right - mid;

        int[] L = new int[n1];
        int[] R = new int[n2];

        for (int i = 0; i < n1; i++)
            L[i] = arr[left + i];
        for (int j = 0; j < n2; j++)
            R[j] = arr[mid + 1 + j];

        int iL = 0, iR = 0;
        int k = left;

        while (iL < n1 && iR < n2)
        {
            if (L[iL] <= R[iR])
                arr[k++] = L[iL++];
            else
                arr[k++] = R[iR++];
        }

        while (iL < n1)
            arr[k++] = L[iL++];
        while (iR < n2)
            arr[k++] = R[iR++];
    }

    static void Main()
    {
        int[] arr = { 38, 27, 43, 3, 9, 82, 10 };
        Console.WriteLine("Before: " + string.Join(", ", arr));

        MergeSort(arr, 0, arr.Length - 1);

        Console.WriteLine("After: " + string.Join(", ", arr));
    }
}
```

👉 Логика:

Разделяем массив на две половины.

Рекурсивно сортируем каждую половину.

Сливаем отсортированные куски в один.

🔹 Quick Sort (Быстрая сортировка)

```csharp
using System;


class Program
{
    static void QuickSort(int[] arr, int low, int high)
    {
        if (low < high)
        {
            int pivotIndex = Partition(arr, low, high);

            // Рекурсивно сортируем левую и правую часть
            QuickSort(arr, low, pivotIndex - 1);
            QuickSort(arr, pivotIndex + 1, high);
        }
    }

    static int Partition(int[] arr, int low, int high)
    {
        int pivot = arr[high]; // опорный элемент
        int i = low - 1;

        for (int j = low; j < high; j++)
        {
            if (arr[j] <= pivot)
            {
                i++;
                Swap(arr, i, j);
            }
        }

        Swap(arr, i + 1, high);
        return i + 1;
    }

    static void Swap(int[] arr, int i, int j)
    {
        int temp = arr[i];
        arr[i] = arr[j];
        arr[j] = temp;
    }

    static void Main()
    {
        int[] arr = { 38, 27, 43, 3, 9, 82, 10 };
        Console.WriteLine("Before: " + string.Join(", ", arr));

        QuickSort(arr, 0, arr.Length - 1);

        Console.WriteLine("After: " + string.Join(", ", arr));
    }
}
```

👉 Логика:

Выбираем опорный элемент (pivot).

Разделяем массив: всё меньше pivot — налево, больше — направо.

Рекурсивно сортируем обе части.

⚡ Разница:

Merge Sort → всегда работает за O(n log n), но требует доп. память.

Quick Sort → среднее O(n log n), но может быть O(n²) в худшем случае (если неудачно выбрать pivot).



---
